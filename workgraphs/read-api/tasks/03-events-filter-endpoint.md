---
id: t_v2filt
slug: events-filter-endpoint
title: GET /events?filter=... — paginated filtered query
workgraph: read-api
order: 3
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
    git ls-remote --heads origin wg-vfobs-read-api/t3-events-filter-endpoint 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T3-1 — GET /events route registered with query params per
    plan §D4: workgraph_id, task_id, agent_id, type, type_namespace,
    org_id (default viloforge — extensibility hook), from_id, limit
    (default 100, capped 1000). Requires Depends(get_read_principal)."
  - "AC-T3-2 — _build_event_query(request) -> EventQuery (Builder
    pattern) maps URL params to a strongly-typed EventQuery
    dataclass; rejects unknown params (422); rejects limit > 1000
    (422); rejects from_id < 0 (422)."
  - "AC-T3-3 — EventsFilterResponse DTO: {v:1,
    events:[{id, event}...], next_from_id: int|None,
    filter_applied: dict}. Each item is a StoredEvent (F1/R2 — the
    T1 read model, serialized directly). filter_applied reflects
    exactly the non-default params the request supplied."
  - "AC-T3-4 — Calls EventRepository.find_filtered (returns
    list[StoredEvent]); serializes StoredEvent directly;
    next_from_id = page[-1].id + 1 if a full page else None."
  - "AC-T3-5 — Unit: tests/unit/test_events_filter.py covers
    EventQuery builder happy path + failure paths (bad limit,
    bad from_id, unknown param via 422 from FastAPI rather than
    silent acceptance), filter_applied shape, next_from_id math."
  - "AC-T3-6 — Integration: tests/integration/test_events_filter_live.py
    seeds events across 3 workgraphs, 4 event types, 2 agents;
    runs combinations of filters and asserts result-set shape +
    pagination via from_id."
  - "AC-T3-7 — Contract: extend tests/contract/test_reads_contract.py
    to include EventsFilterResponse shape."
  - "AC-T3-8 — Branch pushed."

---

# Spec

## Files touched

- `src/vfobs/api/reads.py` (modify — add GET /events route)
- `src/vfobs/api/query.py` (new — EventQuery + _build_event_query)
- `src/vfobs/api/dto.py` (modify — add EventsFilterResponse)
- `tests/unit/test_events_filter.py` (new)
- `tests/integration/test_events_filter_live.py` (new)
- `tests/contract/test_reads_contract.py` (modify)

(NOT touched: repositories (T1 already exposed find_filtered),
auth, adapters, write path.)

## Implementation sketch

> **F1/R2 note:** pseudocode below predates R2. `find_filtered`
> returns `list[StoredEvent]`; serialize StoredEvent directly (no
> `EventDTO`); `next_from_id = page[-1].id + 1`. ACs above are
> authoritative where they differ.

**Patterns: Builder (EventQuery construction; one place where URL
params → typed query object) + Strategy (already in T1's
find_filtered — accepts the EventQuery shape and dispatches to SQL
or list-comprehension).**

```python
from pydantic import BaseModel, Field, ConfigDict

class EventQuery(BaseModel):
    model_config = ConfigDict(frozen=True)
    workgraph_id: str | None = None
    task_id: str | None = None
    agent_id: str | None = None
    type_: str | None = Field(default=None, alias="type")
    type_namespace: str | None = None
    org_id: str | None = None  # request param default applied at handler level
    from_id: int | None = Field(default=None, ge=0)
    limit: int = Field(default=100, ge=1, le=1000)

# FastAPI route — query params are validated by Pydantic via Annotated[...].
# Unknown params return 422 because EventsQueryParams below uses
# extra='forbid'.

@router.get("/events", response_model=EventsFilterResponse)
async def list_events(
    request: Request,
    workgraph_id: str | None = None,
    task_id: str | None = None,
    agent_id: str | None = None,
    type: str | None = None,
    type_namespace: str | None = None,
    org_id: str = "viloforge",
    from_id: int | None = None,
    limit: int = 100,
    principal: Principal = Depends(get_read_principal),
) -> EventsFilterResponse:
    q = EventQuery(
        workgraph_id=workgraph_id, task_id=task_id, agent_id=agent_id,
        type=type, type_namespace=type_namespace, org_id=org_id,
        from_id=from_id, limit=limit,
    )
    repo: EventRepository = request.app.state.event_repo
    events = await repo.find_filtered(
        workgraph_id=q.workgraph_id,
        task_id=q.task_id,
        agent_id=q.agent_id,
        type_=q.type_,
        type_namespace=q.type_namespace,
        org_id=q.org_id,
        from_id=q.from_id,
        limit=q.limit,
    )
    next_from_id = events[-1].id + 1 if len(events) == q.limit else None
    return EventsFilterResponse(
        events=events,
        next_from_id=next_from_id,
        filter_applied=q.model_dump(exclude_none=True, exclude_defaults=True, by_alias=True),
    )
```

## Fail-loud directive (R3)

If any required step (T1 find_filtered behavior, fixture generation,
branch push) cannot be completed: report the failure explicitly.

# Constraints

- limit hard-cap 1000.
- unknown params return 422, never silent accept.
- Pagination by id (not offset) — proper cursor semantics.
- `filter_applied` exposes EXACTLY the supplied non-default params;
  consumers use it to confirm their filter was understood.

# Out of scope

- Time-range filters (v2).
- Order-by other than id-asc.
- Aggregate counts (use /workgraphs/<id>/cost for that, T4).

# References

- plan.md §D4
- T1 EventRepository.find_filtered
- T2 EventDTO
