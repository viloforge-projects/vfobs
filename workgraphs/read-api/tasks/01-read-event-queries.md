---
id: t_v2rqry
slug: read-event-queries
title: EventRepository read-method extension — find_by_* + cost_summary
workgraph: read-api
order: 1
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-read-api/t1-read-event-queries 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T1-1 — EventRepository ABC gains four new async methods per
    plan §D3: find_by_workgraph, find_by_task, find_filtered,
    cost_summary. Existing store + get_by_id signatures unchanged
    (backward-compatible)."
  - "AC-T1-2 — PostgresEventRepository implements all four: SQL
    queries are parameterized (no string interpolation); ordering
    is `id ASC` for find_* and aggregation is server-side for
    cost_summary."
  - "AC-T1-3 — InMemoryEventRepository implements all four with
    list comprehensions + dict reductions; matches Postgres impl
    behavior for the LSP parametrized test."
  - "AC-T1-4 — CostSummary Pydantic model in
    src/vfobs/repositories/event_repository.py exports per plan
    §D3: total_cost_usd: float, total_tokens: int, total_turns: int,
    task_count: int, sample_event_count: int. frozen=True."
  - "AC-T1-5 — find_filtered's order-of-precedence for params:
    workgraph_id > task_id > agent_id > org_id > type/type_namespace
    > from_id > limit. Limit defaults to 100, hard-capped at 1000.
    All params are AND-combined."
  - "AC-T1-6 — cost_summary requires exactly one of workgraph_id or
    agent_id (raises ValueError if both or neither). De-dup
    discipline (verifier F2): for each task_id, count ONLY the
    execution_summary on the highest-id task.state_changed event
    that carries one; tasks with no summary-bearing event
    contribute nothing; a missing field inside an execution_summary
    contributes 0 (no NaN). task_count = number of contributing
    tasks; sample_event_count = number of (post-dedup) contributing
    events. A task with two summary-bearing state_changed events
    (rework) is counted ONCE (the later id wins)."
  - "AC-T1-10 — Stored-event id propagation (verifier F1, mechanism
    R2): the base `Event` is UNCHANGED (its v1 ingest schema is a
    locked contract — adding a field drifts event_schemas.v1.json).
    A read-only `StoredEvent {id:int; event:Event}` model lives in
    event_repository.py; find_by_workgraph/find_by_task/find_filtered
    return list[StoredEvent] (Postgres: StoredEvent(id=row['id'],
    event=_row_to_event(row)); InMemory: StoredEvent(id=eid,
    event=ev)). Unit AC asserts find_* items expose `.id` and
    `.event`, and StoredEvent.model_dump() includes 'id' + nested
    'event' (regression guard for the WG1-F1 silent-drop class +
    proof the locked Event schema is untouched: contract suite
    stays green). _row_to_event, get_by_id, store()->int all
    byte-for-byte unchanged (get_by_id stays bare Event — D-T1-1)."
  - "AC-T1-7 — Unit: tests/unit/test_event_repository.py (extend)
    covers all four methods against InMemoryEventRepository:
    empty store returns empty/zero, mixed events filter correctly,
    cost_summary math is correct for known inputs, ValueError on
    bad cost_summary params."
  - "AC-T1-8 — Integration:
    tests/integration/test_postgres_event_repository.py (extend)
    runs the four methods against real Postgres + same LSP-
    parametrized test pattern from WG1 — both impls satisfy the
    same behavioral contract."
  - "AC-T1-9 — Branch wg-vfobs-read-api/t1-read-event-queries is
    pushed."

---

# Spec

## Files touched

- `src/vfobs/events/base.py` (NOT modified — verifier F1 mechanism
  R2 keeps the locked ingest schema intact; only a clarifying
  comment is acceptable)
- `src/vfobs/repositories/event_repository.py` (modify — extend ABC
  + both impls + add CostSummary + add `StoredEvent` read model;
  `_row_to_event`/`get_by_id` unchanged)
- `src/vfobs/repositories/__init__.py` (modify — export
  `CostSummary` + `StoredEvent`)
- `tests/unit/test_event_repository.py` (modify — extend)
- `tests/integration/test_postgres_event_repository.py` (modify —
  extend)

(NOT touched: any HTTP layer, any auth, any adapter. T1 is pure
data-access layer extension. The `Event.id` field is additive and
frozen-safe; write-path callers are unaffected — stored events keep
`id=None`, the DB/counter assigns, reads reconstruct with `id`.)

## Implementation sketch

**Pattern: Repository extension (additive — no breaking changes to
WG1's interface, no LSP regression).**

### CostSummary model

```python
class CostSummary(BaseModel):
    model_config = ConfigDict(frozen=True)
    total_cost_usd: float
    total_tokens: int
    total_turns: int
    task_count: int
    sample_event_count: int
```

### Postgres SQL sketches

```python
_FIND_BY_WORKGRAPH = sa.text("""
    SELECT id, v, workgraph_id, task_id, agent_id, trace_id,
           source, type, timestamp, data, classification,
           org_id, cluster_id, created_at
    FROM vfobs.events
    WHERE workgraph_id = :workgraph_id
      AND (:from_id IS NULL OR id >= :from_id)
    ORDER BY id ASC
    LIMIT :limit
""")

# find_filtered builds the WHERE clause from non-NULL params.
# All params are AND-combined.

_COST_BY_WORKGRAPH = sa.text("""
    WITH latest AS (
      SELECT DISTINCT ON (task_id)
             task_id,
             data->'execution_summary' AS es
      FROM vfobs.events
      WHERE type = 'task.state_changed'
        AND data ? 'execution_summary'
        AND data->'execution_summary' IS NOT NULL
        AND data->'execution_summary' != 'null'::jsonb
        AND workgraph_id = :workgraph_id
      ORDER BY task_id, id DESC          -- latest summary per task (F2)
    )
    SELECT
      COALESCE(SUM((es->>'cost_usd')::numeric), 0)::float AS total_cost_usd,
      COALESCE(SUM((es->>'total_tokens')::int), 0) AS total_tokens,
      COALESCE(SUM((es->>'num_turns')::int), 0) AS total_turns,
      COUNT(*) AS task_count,
      COUNT(*) AS sample_event_count
    FROM latest
""")
-- task_count == sample_event_count by construction (one row per
-- contributing task post-dedup); both kept in CostSummary for
-- consumer clarity + parity with the agent-scoped query.
```

(R13: every JSONB path traversal works on declared
`ExecutionSummary` fields from T1 of WG1 — `num_turns`,
`total_tokens`, `cost_usd`. No undeclared-key traversal.)

### InMemory impl

```python
async def find_by_workgraph(self, workgraph_id, *, from_id=None, limit=100):
    out = []
    for eid, ev in sorted(self._events.items()):
        if ev.workgraph_id != workgraph_id: continue
        if from_id is not None and eid < from_id: continue
        out.append(StoredEvent(id=eid, event=ev))   # F1/R2 read model
        if len(out) >= min(limit, 1000): break
    return out
# find_by_task / find_filtered wrap StoredEvent the same way.
# (Postgres: StoredEvent(id=row["id"], event=_row_to_event(row)).
#  Nullable from_id cursor is built in Python, NOT a
#  ":p IS NULL OR ..." clause — asyncpg can't type a NULL bigint.)

async def cost_summary(self, *, workgraph_id=None, agent_id=None):
    if (workgraph_id is None) == (agent_id is None):
        raise ValueError("cost_summary requires exactly one of workgraph_id or agent_id")
    # F2: keep the highest-id summary-bearing state_changed per task.
    latest: dict[str, tuple[int, "ExecutionSummary"]] = {}
    for eid, ev in self._events.items():
        if ev.type != "task.state_changed": continue
        if workgraph_id is not None and ev.workgraph_id != workgraph_id: continue
        if agent_id is not None and ev.agent_id != agent_id: continue
        es = ev.data.execution_summary
        if es is None or ev.task_id is None: continue
        cur = latest.get(ev.task_id)
        if cur is None or eid > cur[0]:
            latest[ev.task_id] = (eid, es)
    total_cost, total_tokens, total_turns = 0.0, 0, 0
    for _eid, es in latest.values():
        if es.cost_usd is not None: total_cost += es.cost_usd
        if es.total_tokens is not None: total_tokens += es.total_tokens
        if es.num_turns is not None: total_turns += es.num_turns
    return CostSummary(
        total_cost_usd=total_cost, total_tokens=total_tokens,
        total_turns=total_turns, task_count=len(latest),
        sample_event_count=len(latest),
    )
```

## Fail-loud directive (R3)

If any required step (SQL compatibility issues, LSP test divergence,
branch push) cannot be completed due to missing dependencies,
missing tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success.

# Constraints

- Backward-compatible: existing WG1 callers MUST continue to work
  unmodified. The ABC gains new abstract methods — if any third-
  party impl exists (none yet), it would fail until updated; for
  v1 only InMemory and Postgres impls exist and both are updated
  here.
- LSP-parametrized test from WG1 MUST be extended to cover all
  four new methods.
- Limit param hard-cap = 1000 (prevent runaway queries).
- cost_summary handles NULL execution_summary as 0 — no NaN, no
  error.

# Out of scope

- Endpoint wiring (T2+T3+T4 work).
- Cost-anomaly detection (v2).
- Time-window cost queries (`from`/`to` timestamps) — v2.
- Project-level cost (events lack `project_id` in v1).
- Caching of query results (v2 if needed).

# References

- workgraph.md, plan.md §D3 (this directory).
- `src/vfobs/repositories/event_repository.py` (extended in this task).
- `src/vfobs/events/namespaces/task.py` — TaskStateChangedData +
  ExecutionSummary types this aggregates over.
- `vtf-methodologies/spec-author/bugfix.md` R1-R14.
