---
id: t_R3jL5n
slug: event-repository
title: EventRepository interface + PostgresEventRepository + InMemoryEventRepository
workgraph: foundation
order: 2
required_tags:
  - executor
depends_on:
  - t_K2pQ4r
  - t_M7sV9w
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t2-event-repository 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T2-1 — `src/vfobs/repositories/event_repository.py` defines
    abstract class `EventRepository(ABC)` with two async methods:
    `store(event: Event) -> int` (returns the assigned id) and
    `get_by_id(event_id: int) -> Event | None`. The interface is
    async-only (per plan.md §D4).
  - AC-T2-2 — `PostgresEventRepository(EventRepository)` in the
    same module implements both methods. `store` uses an INSERT
    with `RETURNING id` against `vfobs.events`; serializes the
    typed Event to its column values via a dedicated mapper
    (`_event_to_row(event) -> dict`). `get_by_id` SELECTs by id +
    timestamp range (the PK is composite) and reconstructs the
    typed Event via the discriminated-union the namespaces module
    exposes (`_row_to_event(row) -> Event`).
  - AC-T2-3 — `InMemoryEventRepository(EventRepository)` in
    `src/vfobs/repositories/event_repository.py` implements both
    methods against a plain `list[Event]` plus an `itertools.count`
    counter for ids. Used by unit tests in T2 and downstream
    (T4 unit tests, future WG2-5 unit tests).
  - AC-T2-4 — Unit: `tests/unit/test_event_repository.py` exercises
    `InMemoryEventRepository.store` (returns monotonic ids starting
    at 1), `get_by_id` (round-trips an event), `get_by_id` for an
    absent id (returns None), and the abstract base raises if
    instantiated directly. Pyramid level: unit (no I/O).
  - AC-T2-5 — Integration:
    `tests/integration/test_postgres_event_repository.py` runs
    against a disposable Postgres (same fixture pattern as T0),
    applies the T0 migration, then verifies
    `PostgresEventRepository.store` followed by `get_by_id`
    round-trips an event across the wire including:
    - all base fields (v, workgraph_id, task_id, etc.) preserved
    - `data` JSONB preserved (deep-equals)
    - assigned `id` is a positive integer and monotonically
      increases across two successive `store` calls
    - server-default `created_at` is populated (within 5s of now)
  - AC-T2-6 — Integration:
    `tests/integration/test_postgres_event_repository.py` also
    verifies that storing 17 events across three event types
    yields three distinct id ranges (proving the sequence is
    shared across types — single-table design) and that
    `get_by_id` for each id returns the expected event.
  - AC-T2-7 — Both repos honor the `EventRepository` contract; a
    parametrized test in `tests/unit/test_event_repository.py`
    runs the same behavioral check against both implementations
    (Liskov substitution per engineering principles §3 — the
    interface is the contract, both impls obey it).
  - AC-T2-8 — Branch `wg-vfobs-foundation/t2-event-repository` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `src/vfobs/repositories/__init__.py` (new) — public exports
  (`EventRepository`, `PostgresEventRepository`,
  `InMemoryEventRepository`).
- `src/vfobs/repositories/event_repository.py` (new) — all three
  classes above.
- `src/vfobs/db.py` (new) — SQLAlchemy async engine factory
  (`get_async_engine(settings) -> AsyncEngine`) and `AsyncSession`
  factory. Used by `PostgresEventRepository`. (T3 will also use it.)
- `src/vfobs/events/_dispatch.py` (new) — internal helper:
  `EVENT_TYPE_REGISTRY: dict[str, type[Event]]` populated from the
  factory's registered classes; used by `_row_to_event` to pick
  the right concrete class given the `type` column value.
- `tests/unit/test_event_repository.py` (new).
- `tests/integration/test_postgres_event_repository.py` (new).

(NOT touched: `src/vfobs/api/`, `src/vfobs/main.py`. T2 is the
storage layer only.)

## Implementation sketch

**Patterns: Repository (storage abstraction) + DIP (T4 depends on
the abstract `EventRepository`, not on Postgres). Liskov
Substitution Principle is explicit — `InMemoryEventRepository` is a
real test-time substitute for `PostgresEventRepository`, not a
mock.**

### EventRepository (abstract)

```python
from abc import ABC, abstractmethod
from vfobs.events.base import Event

class EventRepository(ABC):
    @abstractmethod
    async def store(self, event: Event) -> int: ...

    @abstractmethod
    async def get_by_id(self, event_id: int) -> Event | None: ...
```

### PostgresEventRepository

```python
import sqlalchemy as sa
from sqlalchemy.ext.asyncio import AsyncEngine

INSERT_EVENT = sa.text("""
    INSERT INTO vfobs.events (
        v, workgraph_id, task_id, agent_id, trace_id,
        source, type, timestamp, data, classification,
        org_id, cluster_id
    ) VALUES (
        :v, :workgraph_id, :task_id, :agent_id, :trace_id,
        :source, :type, :timestamp, :data, :classification,
        :org_id, :cluster_id
    ) RETURNING id
""")

SELECT_BY_ID = sa.text("""
    SELECT id, v, workgraph_id, task_id, agent_id, trace_id,
           source, type, timestamp, data, classification,
           org_id, cluster_id, created_at
    FROM vfobs.events
    WHERE id = :id
""")

class PostgresEventRepository(EventRepository):
    def __init__(self, engine: AsyncEngine):
        self._engine = engine

    async def store(self, event: Event) -> int:
        row = _event_to_row(event)
        async with self._engine.begin() as conn:
            result = await conn.execute(INSERT_EVENT, row)
            return result.scalar_one()

    async def get_by_id(self, event_id: int) -> Event | None:
        async with self._engine.connect() as conn:
            result = await conn.execute(SELECT_BY_ID, {"id": event_id})
            row = result.mappings().one_or_none()
            return _row_to_event(row) if row else None
```

### Row ↔ Event mappers

```python
import json

def _event_to_row(event: Event) -> dict:
    base = event.model_dump(mode="json")
    # 'data' sub-model is part of the dump; rest of fields map 1:1.
    return {
        "v": base["v"],
        "workgraph_id": base["workgraph_id"],
        "task_id": base.get("task_id"),
        "agent_id": base.get("agent_id"),
        "trace_id": base.get("trace_id"),
        "source": base["source"],
        "type": base["type"],
        "timestamp": event.timestamp,
        "data": json.dumps(base["data"]),
        "classification": base["classification"],
        "org_id": base["org_id"],
        "cluster_id": base["cluster_id"],
    }

def _row_to_event(row: Mapping) -> Event:
    event_cls = EVENT_TYPE_REGISTRY[row["type"]]
    payload = {
        "v": row["v"],
        "type": row["type"],
        "workgraph_id": row["workgraph_id"],
        "task_id": row["task_id"],
        "agent_id": row["agent_id"],
        "trace_id": row["trace_id"],
        "source": row["source"],
        "timestamp": row["timestamp"],
        "classification": row["classification"],
        "org_id": row["org_id"],
        "cluster_id": row["cluster_id"],
        "data": row["data"] if isinstance(row["data"], dict) else json.loads(row["data"]),
    }
    return event_cls.model_validate(payload)
```

### InMemoryEventRepository

```python
import itertools

class InMemoryEventRepository(EventRepository):
    def __init__(self):
        self._events: dict[int, Event] = {}
        self._next_id = itertools.count(1)

    async def store(self, event: Event) -> int:
        eid = next(self._next_id)
        self._events[eid] = event
        return eid

    async def get_by_id(self, event_id: int) -> Event | None:
        return self._events.get(event_id)
```

### EVENT_TYPE_REGISTRY

Populated in `events/_dispatch.py` by iterating `factory.EVENT_CLASSES`
(from T1's `schema.py`) and keying by each class's `type` literal:

```python
from vfobs.events.schema import EVENT_CLASSES

EVENT_TYPE_REGISTRY: dict[str, type[Event]] = {
    cls.model_fields["type"].default: cls for cls in EVENT_CLASSES
}
```

## Fail-loud directive (R3)

If any required step (integration test scaffolding, branch push,
PR creation) cannot be completed due to missing dependencies,
missing tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success. The judge will fail tasks that report
dishonest success; tasks that report failure honestly can be
reworked.

# Constraints

- The interface MUST stay async. Sync-only is a regression.
- Both implementations MUST round-trip an event losslessly (the
  parametrized LSP test verifies this).
- `PostgresEventRepository` MUST NOT cache reads or writes — the
  database is the source of truth and any cache is a future
  optimization (and would invalidate LSP against the in-memory impl).
- `_event_to_row` MUST NOT silently drop fields (e.g., if a v2 event
  adds a column and the row mapping forgets it, integration tests
  must catch it). Achieved by the round-trip integration test.

# Out of scope

- Batch insert (`store_many`). Single-event ingest is enough for
  WG1; batching is a v2 throughput optimization.
- Streaming reads (cursor-based pagination). WG2 work.
- Time-range queries (`events_between(start, end)`). WG2 work.
- Repository-level retention/archival. Delegated to the
  partition-drop worker (out of WG1).
- Caching layers (Redis, etc.). Not in v1.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §C, §D
- Task 00-postgres-schema.md (this directory) for the schema this
  repository targets
- Task 01-event-types.md (this directory) for the Event types this
  repository stores
- SQLAlchemy 2.0 async docs:
  https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
