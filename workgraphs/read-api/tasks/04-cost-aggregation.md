---
id: t_v2cost
slug: cost-aggregation
title: GET /workgraphs/<id>/cost + GET /agents/<id>/cost
workgraph: read-api
order: 4
required_tags:
  - executor
depends_on:
  - t_v2rqry
  - t_v2endp
  - t_v2filt
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-read-api/t4-cost-aggregation 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T4-1 — GET /workgraphs/<id>/cost + GET /agents/<id>/cost
    registered. Both require Depends(get_read_principal).
    Both return WorkgraphCostResponse / AgentCostResponse
    (frozen DTOs with v:1, scope id, summary: CostSummary)."
  - "AC-T4-2 — CostAggregationStrategy(ABC) with two concrete impls:
    ByWorkgraph(workgraph_id) and ByAgent(agent_id). Each calls
    EventRepository.cost_summary with the appropriate kwarg.
    Strategy lives in src/vfobs/api/cost.py."
  - "AC-T4-3 — Unit: tests/unit/test_cost.py covers strategies
    against InMemoryEventRepository — empty store returns zeros,
    one terminal task with execution_summary populates the sum,
    in-flight task (no execution_summary) is excluded, missing
    fields contribute 0 not NaN."
  - "AC-T4-4 — Integration: tests/integration/test_cost_live.py
    against real Postgres — seeds events across two workgraphs +
    two agents with varied execution_summary values, hits both
    endpoints, asserts math matches direct SQL computation."
  - "AC-T4-5 — Contract: extend tests/contract/test_reads_contract.py
    with the two cost response shapes."
  - "AC-T4-6 — 404 from workgraphs/<id>/cost iff the workgraph_id
    has no events at all in vfobs (regardless of vtaskforge state
    — cost is a pure vfobs view). Same logic for agents/<id>/cost."
  - "AC-T4-7 — Branch pushed."

---

# Spec

## Files touched

- `src/vfobs/api/cost.py` (new — strategies + handlers)
- `src/vfobs/api/reads.py` (modify — register the two cost routes
  OR import from cost.py — either pattern is fine; pick whichever
  reads cleanest)
- `src/vfobs/api/dto.py` (modify — add WorkgraphCostResponse,
  AgentCostResponse)
- `tests/unit/test_cost.py` (new)
- `tests/integration/test_cost_live.py` (new)
- `tests/contract/test_reads_contract.py` (modify)

## Implementation sketch

**Pattern: Strategy (CostAggregationStrategy is a small interface
with two impls; one per aggregation scope). Repository (delegates
to T1's cost_summary).**

```python
class CostAggregationStrategy(ABC):
    @abstractmethod
    async def compute(self, repo: EventRepository) -> CostSummary: ...

class ByWorkgraph(CostAggregationStrategy):
    def __init__(self, workgraph_id: str):
        self._workgraph_id = workgraph_id
    async def compute(self, repo: EventRepository) -> CostSummary:
        return await repo.cost_summary(workgraph_id=self._workgraph_id)

class ByAgent(CostAggregationStrategy):
    def __init__(self, agent_id: str):
        self._agent_id = agent_id
    async def compute(self, repo: EventRepository) -> CostSummary:
        return await repo.cost_summary(agent_id=self._agent_id)

@router.get("/workgraphs/{workgraph_id}/cost", response_model=WorkgraphCostResponse)
async def workgraph_cost(workgraph_id, request, principal: Principal = Depends(get_read_principal)):
    repo = request.app.state.event_repo
    # 404 check: any events for this workgraph?
    if not await repo.find_by_workgraph(workgraph_id, limit=1):
        raise HTTPException(404, "no events for workgraph")
    summary = await ByWorkgraph(workgraph_id).compute(repo)
    return WorkgraphCostResponse(workgraph_id=workgraph_id, summary=summary)
```

(R13: all CostSummary fields are declared in the model; no
undeclared-key mutation. R14: no external chart/image dependencies
in this task — pure code.)

## Fail-loud directive (R3)

If any required step (T1 cost_summary behavior, contract fixture,
branch push) cannot be completed: report the failure explicitly.

# Constraints

- Cost is computed on-the-fly per request — no caching in v1
  (events table is the source of truth).
- Aggregations are NULL-tolerant; an in-flight task does not break
  the rollup.
- v1 ignores project-level scope (events don't carry project_id).

# Out of scope

- Per-project cost rollups (v2 — needs event-schema bump).
- Time-window cost (v2).
- Burn-rate / streaming cost (v2 — needs SSE).
- Cost-anomaly detection (v2 worker).

# References

- plan.md §D4 (cost endpoint shapes) + §D5 (computation discipline)
- T1 EventRepository.cost_summary
- WG1 task.state_changed event with ExecutionSummary payload
