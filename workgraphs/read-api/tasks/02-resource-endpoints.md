---
id: t_v2endp
slug: resource-endpoints
title: GET /workgraphs/<id> + /tasks/<id> + /tasks/<id>/events
workgraph: read-api
order: 2
required_tags:
  - executor
depends_on:
  - t_v2adp1
  - t_v2rqry
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-read-api/t2-resource-endpoints 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T2-1 — src/vfobs/api/reads.py registers three FastAPI
    routes per plan §D4: GET /workgraphs/<id>, GET /tasks/<id>,
    GET /tasks/<id>/events. All require Depends(get_read_principal)
    — vtf-token piggyback auth from T0."
  - "AC-T2-2 — GET /workgraphs/<id> returns the response shape from
    plan §D4: {v:1, workgraph_id, vtf:{...projected metadata...},
    vfobs:{event_count, last_event_id, last_event_type,
    last_event_at}}. Composes T0's VtfClient.get_workgraph + a
    T2-internal private `_find_last_by_workgraph(workgraph_id)`
    helper (T1's find_by_workgraph fixes `id ASC` and exposes no
    order param — see plan §D3; the helper does
    `... ORDER BY id DESC LIMIT 1`). `last_event_id` reads
    `event.id`, which is populated per verifier F1 (Event gains a
    declared `id`). 404 if vtf returns None AND vfobs has no events
    for the id."
  - "AC-T2-3 — GET /tasks/<id> returns the equivalent task-scoped
    shape; same 404 logic against VtfClient.get_task + T1's
    find_by_task."
  - "AC-T2-4 — GET /tasks/<id>/events returns paginated event log:
    {v:1, task_id, events:[...Event JSON...], next_from_id: int|None}.
    Query params: from_id (optional), limit (default 100, capped
    1000). next_from_id is last event's id + 1 if a full page was
    returned else None."
  - "AC-T2-5 — Response DTOs declared in src/vfobs/api/dto.py
    (frozen Pydantic models): WorkgraphReadResponse,
    TaskReadResponse, TaskEventsResponse, EventDTO. EventDTO is
    Event.model_dump'able — no separate transform layer."
  - "AC-T2-6 — Unit: tests/unit/test_reads.py covers each endpoint
    with stubbed VtfClient + InMemoryEventRepository + a
    StaticPrincipalAuth — happy path returns 200 + shape, missing
    resource returns 404, no token returns 401."
  - "AC-T2-7 — Integration: tests/integration/test_reads_live.py
    against real Postgres + a fake vtaskforge ASGI app (httpx
    AsyncClient mock transport) — seeds 5 events for a workgraph,
    hits /workgraphs/<id> + /tasks/<id>/events, asserts payload
    matches seeded data."
  - "AC-T2-8 — Contract: tests/contract/test_reads_contract.py
    pins the response shape against a committed JSON fixture
    (analogous to WG1's event-schema fixture pattern)."
  - "AC-T2-9 — Branch pushed."

---

# Spec

## Files touched

- `src/vfobs/api/reads.py` (new — three GET handlers)
- `src/vfobs/api/dto.py` (new — response Pydantic models)
- `src/vfobs/main.py` (modify — include the new router)
- `tests/unit/test_reads.py` (new)
- `tests/integration/test_reads_live.py` (new)
- `tests/contract/test_reads_contract.py` (new)
- `tests/fixtures/reads_response_shapes.v1.json` (new — contract fixture)

(NOT touched: existing WG1 write path; auth.py (T0 adds read_auth.py);
events/.)

## Implementation sketch

**Pattern: composes Adapter (T0 VtfClient) + Repository (T1
EventRepository) behind the ReadAuth Strategy dep. No new pattern
introduced — composition is enough.**

```python
from fastapi import APIRouter, Depends, HTTPException, Request, status
from vfobs.api.read_auth import get_read_principal, Principal
from vfobs.api.dto import (
    WorkgraphReadResponse, VtfPart, VfobsPart, TaskReadResponse,
    TaskEventsResponse, EventDTO,
)

router = APIRouter()

@router.get("/workgraphs/{workgraph_id}", response_model=WorkgraphReadResponse)
async def get_workgraph(
    workgraph_id: str,
    request: Request,
    principal: Principal = Depends(get_read_principal),
) -> WorkgraphReadResponse:
    vtf: VtfClient = request.app.state.vtf_client
    repo: EventRepository = request.app.state.event_repo
    token = _extract_bearer(request)  # passthrough; ReadAuth already verified
    vtf_meta = await vtf.get_workgraph(workgraph_id, token)
    events_tail = await repo.find_by_workgraph(workgraph_id, limit=1)
    if vtf_meta is None and not events_tail:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "workgraph not found")
    return WorkgraphReadResponse(
        workgraph_id=workgraph_id,
        vtf=VtfPart.from_metadata(vtf_meta) if vtf_meta else None,
        vfobs=VfobsPart.from_tail(events_tail, count=await _count(repo, workgraph_id)),
    )
```

(For "last event" we technically want desc-order limit 1; pragmatic
implementation: query a tail via find_by_workgraph with a large
from_id then take the last result, OR add a dedicated `find_last`
helper to T1 — defer that micro-optimization, the v1 event volume
makes find_by_workgraph(limit=1) acceptable.)

Wait — find_by_workgraph returns from oldest to newest ordering by
plan §D3. For "last event" we need the highest id. Two options:

(a) Add `_find_last_by_workgraph(workgraph_id) -> Event | None` as
    a private helper on the repo. Best, cheapest.
(b) Add `order: Literal["asc","desc"]` param to find_by_workgraph.
    Pollutes interface a little.

Pick (a): keep T1's interface clean; T2 adds the helper as a
PRIVATE method during impl. The helper does:
`SELECT * FROM vfobs.events WHERE workgraph_id=:id ORDER BY id DESC LIMIT 1`.
Document in T2 spec body that this is a T2-internal helper; T1's
ABC stays the four declared methods.

Similar for /tasks/<id>: `_find_last_by_task(task_id)`.

For `_count(repo, workgraph_id)`: another T2-internal helper that
SELECTs `COUNT(*) WHERE workgraph_id=:id`.

These small T2-internal helpers MUST be tested at unit level (their
own LSP-parametrized test against both repos).

## Fail-loud directive (R3)

If any required step (T0/T1 dependencies not landed, contract
fixture generation, branch push) cannot be completed: report the
failure explicitly in your completion notes — do not rationalize
partial completion as success.

# Constraints

- All three endpoints require auth; no public reads in v1.
- 404 only when BOTH vtaskforge AND vfobs have nothing on the id —
  asymmetric data (vfobs has events, vtaskforge purged the record,
  or vice versa) returns 200 with the present half populated and
  the missing half as null.
- Pagination cap: limit=1000 hard maximum on /tasks/<id>/events.
- Response DTOs are version 1 (`v: 1`); future shape changes bump v.

# Out of scope

- Filtering on the per-resource endpoints (use /events?filter=... for that).
- Time-window queries.
- Aggregate stats beyond event_count + last-event tail.
- SSE on these endpoints (WG3).

# References

- workgraph.md, plan.md §D4 (this directory).
- T0 `src/vfobs/adapters/vtf.py` — VtfClient + DTOs.
- T1 `src/vfobs/repositories/event_repository.py` — new read
  methods this task composes.
- `vtf-methodologies/spec-author/bugfix.md` R1-R14.
