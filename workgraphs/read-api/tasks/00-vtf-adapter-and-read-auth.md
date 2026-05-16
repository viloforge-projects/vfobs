---
id: t_v2adp1
slug: vtf-adapter-and-read-auth
title: Vtaskforge adapter + ReadAuth Strategy (vtf-token piggyback)
workgraph: read-api
order: 0
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-read-api/t0-vtf-adapter-and-read-auth 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T0-1 — src/vfobs/adapters/vtf.py defines VtfClient with httpx
    async impl: async whoami(token) -> WhoamiPrincipal, async
    get_workgraph(workgraph_id, token) -> WorkgraphMetadata | None,
    async get_task(task_id, token) -> TaskMetadata | None. Base URL
    from Settings.vtaskforge_url; timeout from
    Settings.vtaskforge_timeout_seconds (default 5)."
  - "AC-T0-2 — Pydantic models WhoamiPrincipal, WorkgraphMetadata,
    TaskMetadata in src/vfobs/adapters/dto.py — project to the
    vfobs-visible fields only (per plan §D2). frozen=True."
  - "AC-T0-3 — src/vfobs/api/read_auth.py defines ReadAuth(ABC) with
    async verify(token: str | None) -> Principal (raises 401 on
    failure). VtfTokenAuth(ReadAuth) implementation: sha256 hex-prefix
    16 of the token → TTL cache (60s) → on miss call
    VtfClient.whoami(token). Cache uses dict + monotonic clock; no
    third-party cache library. Raw token NEVER logged."
  - "AC-T0-4 — StaticPrincipalAuth(ReadAuth) test substitute returns
    a fixed principal regardless of token. Used by non-auth-focused
    tests in WG2."
  - "AC-T0-5 — src/vfobs/main.py extends create_app to accept
    read_auth=None and vtf_client=None kwargs (test injection
    parity with WG1). lifespan resolves production
    VtfClient(settings) + VtfTokenAuth(vtf_client) when not injected."
  - "AC-T0-6 — Settings (src/vfobs/config.py) gains:
    vtaskforge_url: HttpUrl (no default — must be set in prod) and
    vtaskforge_timeout_seconds: int = 5. Existing fields unchanged."
  - "AC-T0-7 — Unit: tests/unit/test_vtf_adapter.py exercises
    VtfClient against an httpx MockTransport: whoami 200 → returns
    Principal, 401 → raises a typed VtfAuthError; get_workgraph
    200 → projects to WorkgraphMetadata, 404 → returns None;
    get_task same shape."
  - "AC-T0-8 — Unit: tests/unit/test_read_auth.py covers VtfTokenAuth
    cache behavior: cold miss calls whoami (verify via stubbed
    VtfClient), warm hit does NOT call whoami again, post-TTL
    expiry re-fetches, raw token does not appear in cache keys
    (only the sha256 prefix), 401 from upstream maps to 401 here.
    StaticPrincipalAuth returns the configured principal regardless
    of token."
  - "AC-T0-9 — Integration: tests/integration/test_vtf_adapter_live.py
    spins up an httpx-test ASGI app that mimics vtaskforge's whoami
    + workgraph endpoints; VtfClient hits it via real network on
    a unix socket or localhost ASGI transport; verifies the
    full request shape (Authorization header passthrough, URL
    composition, timeout config)."
  - "AC-T0-10 — Branch wg-vfobs-read-api/t0-vtf-adapter-and-read-auth
    is pushed to viloforge/vfobs and a PR is open."

---

# Spec

## Files touched

- `src/vfobs/adapters/__init__.py` (new — package marker)
- `src/vfobs/adapters/vtf.py` (new — VtfClient httpx impl)
- `src/vfobs/adapters/dto.py` (new — Pydantic models)
- `src/vfobs/api/read_auth.py` (new — ReadAuth + VtfTokenAuth +
  StaticPrincipalAuth + get_read_principal dependency)
- `src/vfobs/config.py` (modify — add vtaskforge_url +
  vtaskforge_timeout_seconds)
- `src/vfobs/main.py` (modify — add read_auth, vtf_client kwargs;
  lifespan resolves production pair)
- `tests/unit/test_vtf_adapter.py` (new)
- `tests/unit/test_read_auth.py` (new)
- `tests/integration/test_vtf_adapter_live.py` (new)

(NOT touched: existing src/vfobs/api/auth.py — that stays for the
write path; src/vfobs/events/, src/vfobs/repositories/, charts/,
existing tests.)

## Implementation sketch

**Patterns: Adapter (T0 wraps vtaskforge's HTTP API behind a typed
interface) + Strategy (ReadAuth interface admits multiple impls;
v1 ships VtfTokenAuth + a test substitute).**

### dto.py

```python
from pydantic import BaseModel, ConfigDict

class WhoamiPrincipal(BaseModel):
    model_config = ConfigDict(frozen=True)
    user_id: str
    display_name: str | None = None

class WorkgraphMetadata(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    status: str
    kind: str | None = None
    target_repos: list[str] = []
    tags: list[str] = []
    created_at: str | None = None  # ISO 8601

class TaskMetadata(BaseModel):
    model_config = ConfigDict(frozen=True)
    id: str
    workgraph_id: str | None = None
    status: str
    title: str | None = None
    created_at: str | None = None
```

(R13: every key on these models is declared; no `model_copy(update={...})`
of undeclared fields anywhere downstream.)

### VtfClient (Adapter)

```python
class VtfAuthError(Exception): ...

class VtfClient:
    def __init__(self, settings: Settings, http: httpx.AsyncClient | None = None):
        self._base = str(settings.vtaskforge_url).rstrip("/")
        self._timeout = settings.vtaskforge_timeout_seconds
        self._http = http or httpx.AsyncClient(timeout=self._timeout)

    async def whoami(self, token: str) -> WhoamiPrincipal:
        r = await self._http.get(f"{self._base}/v2/auth/whoami",
                                 headers={"Authorization": f"Bearer {token}"})
        if r.status_code == 401:
            raise VtfAuthError("invalid vtf token")
        r.raise_for_status()
        return WhoamiPrincipal.model_validate(r.json())

    async def get_workgraph(self, workgraph_id: str, token: str) -> WorkgraphMetadata | None:
        r = await self._http.get(
            f"{self._base}/v2/workgraphs/{workgraph_id}/",
            headers={"Authorization": f"Bearer {token}"},
        )
        if r.status_code == 404:
            return None
        r.raise_for_status()
        return WorkgraphMetadata.model_validate(r.json())

    async def get_task(self, task_id: str, token: str) -> TaskMetadata | None:
        # symmetric impl
        ...

    async def aclose(self) -> None:
        await self._http.aclose()
```

### read_auth.py

```python
import hashlib, time
from abc import ABC, abstractmethod
from dataclasses import dataclass
from fastapi import Depends, HTTPException, Request, status

@dataclass(frozen=True)
class Principal:
    user_id: str
    display_name: str | None = None

class ReadAuth(ABC):
    @abstractmethod
    async def verify(self, token: str | None) -> Principal: ...

def _hash_prefix(token: str) -> str:
    return hashlib.sha256(token.encode()).hexdigest()[:16]

class VtfTokenAuth(ReadAuth):
    def __init__(self, vtf: VtfClient, ttl_seconds: int = 60):
        self._vtf = vtf
        self._ttl = ttl_seconds
        self._cache: dict[str, tuple[Principal, float]] = {}

    async def verify(self, token: str | None) -> Principal:
        if not token:
            raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Missing bearer token")
        key = _hash_prefix(token)
        now = time.monotonic()
        cached = self._cache.get(key)
        if cached and (now - cached[1]) < self._ttl:
            return cached[0]
        try:
            principal = await self._vtf.whoami(token)
        except VtfAuthError:
            raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid bearer token")
        p = Principal(user_id=principal.user_id, display_name=principal.display_name)
        self._cache[key] = (p, now)
        return p

class StaticPrincipalAuth(ReadAuth):
    def __init__(self, principal: Principal):
        self._principal = principal

    async def verify(self, token: str | None) -> Principal:
        return self._principal
```

`get_read_principal` FastAPI dep mirrors the write-side
`get_principal` shape; resolves `app.state.read_auth.verify(...)`.

### main.py extension

```python
def create_app(
    settings=None, *, event_repo=None, ingest_auth=None,
    read_auth=None, vtf_client=None,
) -> FastAPI:
    ...
    app.state.read_auth = read_auth
    app.state.vtf_client = vtf_client

@asynccontextmanager
async def _lifespan(app):
    ...
    # F-adv4: match the existing main.py:20-23 idiom (hasattr/is None),
    # not getattr-with-default, so the test-injection parity is exact.
    if not hasattr(app.state, "vtf_client") or app.state.vtf_client is None:
        app.state.vtf_client = VtfClient(app.state.settings)
    if not hasattr(app.state, "read_auth") or app.state.read_auth is None:
        app.state.read_auth = VtfTokenAuth(app.state.vtf_client)
    try:
        yield
    finally:
        ...
        if app.state.vtf_client is not None:
            await app.state.vtf_client.aclose()
```

## Fail-loud directive (R3)

If any required step (vtaskforge mock setup, branch push, PR
creation) cannot be completed due to missing dependencies, missing
tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success.

# Constraints

- The raw vtf-token MUST NEVER appear in cache keys, log records,
  or error messages. SHA256-prefix is the only on-wire form
  outside the original `Authorization` header.
- `VtfClient` MUST timeout at `Settings.vtaskforge_timeout_seconds`;
  a hung vtaskforge cannot block vfobs forever.
- 401 from vtaskforge maps to 401 from vfobs — no leaking upstream
  status codes (e.g., 5xx from vtaskforge → 503 from vfobs, not
  401 which would be misleading; that's a v2 detail though).
- `Settings.vtaskforge_url` MUST be set; missing → app fails to
  start (Pydantic validation catches this).
- v1 cache is process-local; multi-replica deployments need a
  shared cache. Out of scope for v1 (single replica).

# Out of scope

- Multi-replica cache coordination (v2).
- Token rotation handling beyond TTL expiry.
- OIDC / web-user auth.
- vtaskforge schema mirroring beyond the projected metadata fields.
- Rate limiting at vfobs side.

# References

- workgraph.md, plan.md (this directory) — design and DAG.
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §I.
- `vtf-methodologies/spec-author/bugfix.md` R1-R14 — applied,
  especially R13 (no undeclared-key mutations on the projected
  Pydantic models).
- WG1 `src/vfobs/api/auth.py` — sibling Strategy for the write side;
  this task's ReadAuth follows the same shape.
