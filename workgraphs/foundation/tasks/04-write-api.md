---
id: t_W4kF6h
slug: write-api
title: POST /events — Chain of Responsibility ingest pipeline + Strategy-based auth
workgraph: foundation
order: 4
required_tags:
  - executor
depends_on:
  - t_M7sV9w
  - t_R3jL5n
  - t_T8xY1c
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t4-write-api 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T4-1 — `src/vfobs/api/auth.py` defines `IngestAuth(ABC)` with
    one abstract method `verify(token: str | None) -> str` returning
    the resolved principal name on success, raising
    `fastapi.HTTPException(401)` on failure. Concrete impl
    `StaticTokenAuth(IngestAuth)` reads the canonical token from
    Settings and compares constant-time
    (`hmac.compare_digest`). On match, returns
    principal `"controller"`. (Strategy pattern locked here.)
  - AC-T4-2 — `src/vfobs/api/events.py` defines `POST /events` that:
    (a) requires `Authorization: Bearer <token>` header — missing
        or malformed → 401 via the auth dependency
    (b) accepts a request body that is a discriminated union of
        all v1 event types (the union from T1's namespaces) using
        Pydantic's `Discriminator` on the `type` field
    (c) executes the Chain of Responsibility pipeline (validator
        already ran via FastAPI, then enricher, then storer)
    (d) returns `201 Created` with body `{"id": <int>}` on success
    (e) returns 422 on invalid event shape (handled by FastAPI)
    (f) returns 500 with logged error on storage failure
  - AC-T4-3 — `src/vfobs/api/events.py` factors the pipeline into
    named handlers: `Enricher` (Chain of Responsibility node 1)
    and `Storer` (CoR node 2). Each is a plain async callable
    taking `(event: Event, context: dict) -> Event`. The handler
    chain is composed in `_ingest_pipeline()` and called from the
    POST handler. Validator is implicit (FastAPI handles it).
  - AC-T4-4 — `Enricher` adds (when missing) a `server_received_at`
    timestamp into `event.data` via a model-rebuild helper
    (`event.model_copy(update={"data": {**event.data.model_dump(),
    "server_received_at": now()}})`). Source is left as-supplied by
    the controller (DESIGN §C — controllers self-identify as their
    source). The Enricher is the single place server-time
    augmentation happens; future enrichments (geo, tenant tagging)
    add nodes here without touching the storer or validator.
  - AC-T4-5 — `Storer` calls `EventRepository.store(event)` (the
    repo is resolved from app state — `request.app.state.event_repo`,
    set in `create_app`'s lifespan from T3) and returns the
    same event annotated with the assigned id via `context["id"]`.
  - AC-T4-6 — Unit: `tests/unit/test_post_events.py` constructs the
    app with an `InMemoryEventRepository` and a stubbed Settings
    (token="testtok"). Covers: valid event POST → 201 with id=1;
    valid event but wrong token → 401; missing token → 401; bad
    event shape → 422; the second valid POST returns id=2;
    Enricher injects `server_received_at` (visible in the
    repository's stored event). Pyramid level: unit.
  - AC-T4-7 — Integration: `tests/integration/test_post_events_live.py`
    spins the app against a disposable Postgres (T0 fixture
    pattern) wired to `PostgresEventRepository`, POSTs a real
    event over HTTP via `httpx.AsyncClient`, asserts 201 + id, then
    queries Postgres directly to verify the event is in
    `vfobs.events` with `server_received_at` set.
  - AC-T4-8 — Contract: `tests/contract/test_post_events_contract.py`
    posts one event of each of the 17 event types defined in T1
    using EventFactory builders and asserts each 201s with a
    monotonically increasing id. Drift between T1's types and
    what the write API accepts is caught here.
  - AC-T4-9 — Pipeline ordering test: a stub `Enricher` substitute
    that records its call order proves
    Enricher runs strictly before Storer; an exception in Enricher
    short-circuits the chain (Storer not called). Unit-level test.
  - AC-T4-10 — Branch `wg-vfobs-foundation/t4-write-api` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `src/vfobs/api/auth.py` (new) — `IngestAuth`, `StaticTokenAuth`,
  `get_ingest_auth()` FastAPI dependency.
- `src/vfobs/api/events.py` (new) — POST /events handler +
  Chain of Responsibility pipeline.
- `src/vfobs/main.py` (modify) — mount events router; resolve
  `EventRepository` into `app.state.event_repo` in lifespan
  (PostgresEventRepository(app.state.engine) on startup);
  resolve `IngestAuth` into `app.state.ingest_auth` similarly.
- `tests/unit/test_post_events.py` (new).
- `tests/integration/test_post_events_live.py` (new).
- `tests/contract/test_post_events_contract.py` (new).

(NOT touched: anything outside `src/vfobs/api/`,
`src/vfobs/main.py`. T4 wires existing T1/T2/T3 pieces; no new
storage or schema work.)

## Implementation sketch

**Patterns: Chain of Responsibility (validator → enricher → storer
pipeline; each node mutates context and can short-circuit) +
Strategy (`IngestAuth` interface with `StaticTokenAuth` impl,
swappable for per-pod tokens or mTLS in v2).**

### auth.py

```python
import hmac
from abc import ABC, abstractmethod
from fastapi import Depends, HTTPException, Request, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from vfobs.config import Settings

bearer = HTTPBearer(auto_error=False)

class IngestAuth(ABC):
    @abstractmethod
    def verify(self, token: str | None) -> str: ...

class StaticTokenAuth(IngestAuth):
    def __init__(self, settings: Settings):
        self._expected = settings.ingest_token.get_secret_value()

    def verify(self, token: str | None) -> str:
        if not token:
            raise HTTPException(status.HTTP_401_UNAUTHORIZED,
                                "Missing bearer token")
        if not hmac.compare_digest(token, self._expected):
            raise HTTPException(status.HTTP_401_UNAUTHORIZED,
                                "Invalid bearer token")
        return "controller"

async def get_principal(
    request: Request,
    creds: HTTPAuthorizationCredentials | None = Depends(bearer),
) -> str:
    auth: IngestAuth = request.app.state.ingest_auth
    return auth.verify(creds.credentials if creds else None)
```

### events.py — Chain of Responsibility

```python
from datetime import datetime, UTC
from typing import Annotated, Union
from pydantic import Field
from fastapi import APIRouter, Depends, Request, status
from vfobs.events.namespaces import (
    workgraph as wg, task, harness, gate, judge, anomaly,
)
from vfobs.api.auth import get_principal
from vfobs.events.base import Event

router = APIRouter()

# Discriminated union — Pydantic dispatches by 'type' literal.
AnyEvent = Annotated[
    Union[
        wg.WorkgraphCreated, wg.WorkgraphStateChanged, wg.WorkgraphCompleted,
        task.TaskClaimed, task.TaskStateChanged, task.TaskHeartbeat,
        task.TaskWorkdirChanged,
        harness.HarnessTurnStarted, harness.HarnessToolCall,
        harness.HarnessTurnCompleted,
        gate.GateStarted, gate.GatePassed, gate.GateFailed,
        judge.JudgeStarted, judge.JudgeReviewSubmitted, judge.JudgeDecision,
        anomaly.AnomalyStuckDetected,
    ],
    Field(discriminator="type"),
]

# CoR node 1
async def enricher(event: Event, context: dict) -> Event:
    data = event.data.model_copy(update={
        "server_received_at": context.get("server_received_at",
                                          datetime.now(UTC)),
    })
    return event.model_copy(update={"data": data})

# CoR node 2
async def storer(event: Event, context: dict) -> Event:
    repo = context["repo"]
    eid = await repo.store(event)
    context["id"] = eid
    return event

PIPELINE = [enricher, storer]

async def _run_pipeline(event: Event, context: dict) -> int:
    current = event
    for node in PIPELINE:
        current = await node(current, context)
    return context["id"]

@router.post("/events", status_code=status.HTTP_201_CREATED)
async def post_events(
    request: Request,
    body: AnyEvent,
    principal: str = Depends(get_principal),
) -> dict:
    context = {
        "repo": request.app.state.event_repo,
        "principal": principal,
        "server_received_at": datetime.now(UTC),
    }
    eid = await _run_pipeline(body, context)
    return {"id": eid}
```

### Wiring in main.py (extends T3)

```python
async def lifespan(app: FastAPI):
    engine: AsyncEngine = build_engine(app.state.settings)
    app.state.engine = engine
    app.state.event_repo = PostgresEventRepository(engine)
    app.state.ingest_auth = StaticTokenAuth(app.state.settings)
    try:
        yield
    finally:
        await engine.dispose()
```

### Note on data-field augmentation

The Enricher adds `server_received_at` into `event.data` because the
base Event schema (T1) deliberately doesn't carry a server-time
field — keeps T1's pure-data discipline. Storing
`server_received_at` in `data` means it's preserved through the
JSONB column and visible to consumers. The base table column
`created_at` (T0) is the row-insert audit timestamp; they're
semantically different and both useful.

## Fail-loud directive (R3)

If any required step (test scaffolding, integration deployment,
branch push) cannot be completed due to missing dependencies,
missing tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success. The judge will fail tasks that report
dishonest success; tasks that report failure honestly can be
reworked.

# Constraints

- Auth comparison MUST be constant-time (`hmac.compare_digest`).
  A regular `==` would expose token-length information through
  timing.
- The pipeline's order is locked: validation (FastAPI) → enrichment →
  storage. Storage MUST NOT happen before enrichment; enrichment
  MUST NOT happen before validation.
- Storage failures bubble up as 500. Do NOT swallow them and return
  200 — that would create a ghost-write defect (analogous to the
  ghost-completion class the canary spike found).
- `server_received_at` MUST be set to **server time**, not
  client-supplied. Used for storage audit, ordering ambiguity
  resolution, and downstream anomaly detection in WG4.
- No batching, no async queuing, no fire-and-forget. POST blocks
  until storage succeeds and returns the id.

# Out of scope

- Rate limiting / quotas. v2.
- Idempotency keys (controller retries duplicating events). v2 —
  acceptable in v1 because v1's controller doesn't retry.
- Token rotation. Operator-managed via Secret update in v1.
- Per-pod service-account tokens. v2 — same Strategy interface, new
  `MultiTokenAuth` impl.
- mTLS for write-API auth. v2 — same Strategy interface, new
  `MutualTLSAuth` impl.
- OIDC / vtf-token piggyback (the **read** half of OIQ3). WG2.
- Async event publishing to in-process subscribers. WG3 work; T4
  only persists.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §C, §F, §I
- Pydantic discriminated unions:
  https://docs.pydantic.dev/latest/concepts/unions/#discriminated-unions
- FastAPI HTTPBearer:
  https://fastapi.tiangolo.com/reference/security/?h=httpbearer#fastapi.security.HTTPBearer
- Task 01-event-types.md (this directory) — event union shapes
- Task 02-event-repository.md (this directory) — `EventRepository.store`
- Task 03-fastapi-skeleton.md (this directory) — `create_app`,
  middleware, settings
