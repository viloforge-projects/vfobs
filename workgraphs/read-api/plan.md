---
authored_by: claude-opus-4-7
co_authors:
  - operator:lonvilo
created_at: 2026-05-14T00:00:00Z
last_revision: 2026-05-14T00:00:00Z
---

# Approach

WG1 shipped a working event-write path; WG2 adds the read side.
The shape is: one adapter for vtaskforge resource metadata
(workgraph/task records), a ReadAuth Strategy that pigggybacks
on vtf-token validation, a small set of EventRepository read
methods that operate over the existing time-partitioned events
table, and three composed HTTP endpoints. Cost rollups are
computed on the fly from `data.execution_summary` payloads
already in storage (no schema changes required).

# Design decisions

## D1 — Read auth (OIQ3 resolution, read-half): vtf-token piggyback with hash-cached whoami

Per DESIGN §I, operator-facing reads reuse the vtaskforge bearer
token. The `ReadAuth.verify(token)` call:

1. Hashes the token (sha256, hex prefix 16 chars) — never persists or
   logs the raw token.
2. Looks up the hash in an in-memory TTL cache (60s). Hit → return
   the cached principal.
3. Miss → call `GET <vtaskforge>/v2/auth/whoami` with the token; on
   200 cache `{hash → principal}` for 60s and return; on 401 raise
   `HTTPException(401)`.

Strategy: `ReadAuth(ABC)`; concrete `VtfTokenAuth(ReadAuth)`. The
TTL cache lives in the strategy instance — process-local; multi-pod
scale (v2+) needs Redis or a sidecar, but v1 is single replica per
WG1 PDB design. Test substitute is `StaticPrincipalAuth(ReadAuth)`
that returns a fixed principal for any token (used by all
non-auth-focused tests).

Why this and not a separate vfobs token store:

- Operators already have vtf tokens; making them juggle a vfobs
  token doubles the credential surface for no gain.
- vtaskforge is the authority on operator identity; vfobs deferring
  is correct separation of concerns.
- The 60s cache amortizes the whoami call cost ~99% across a normal
  CLI session — back-of-envelope: a CLI poll every 1s for 5 minutes
  is 5 whoami calls per token, not 300.

## D2 — Vtaskforge adapter (for resource metadata)

`VtfClient` (Adapter pattern) — a thin httpx wrapper that exposes:

- `whoami(token)` → `WhoamiPrincipal`
- `get_workgraph(workgraph_id)` → `WorkgraphMetadata | None`
- `get_task(task_id)` → `TaskMetadata | None`

Implementations in `src/vfobs/adapters/vtf.py`. `WorkgraphMetadata`
and `TaskMetadata` are typed Pydantic models capturing only the
fields vfobs's read endpoints need (id, status, kind, tags, repos,
created_at). vfobs does NOT mirror vtaskforge's full schema — adapter
projects to the fields we expose.

Config: `Settings.vtaskforge_url` (new field), `Settings.vtaskforge_timeout_seconds`
(default 5). No vfobs-to-vtf credentials needed for the metadata
endpoints — they accept the same vtf-token the inbound request
carried (passthrough).

## D3 — EventRepository read-method extension

Additive only. New abstract methods on `EventRepository`:

```python
async def find_by_workgraph(
    self, workgraph_id: str, *,
    from_id: int | None = None, limit: int = 100,
) -> list[Event]: ...

async def find_by_task(
    self, task_id: str, *,
    from_id: int | None = None, limit: int = 100,
) -> list[Event]: ...

async def find_filtered(
    self, *,
    workgraph_id: str | None = None,
    task_id: str | None = None,
    agent_id: str | None = None,
    type_: str | None = None,
    type_namespace: str | None = None,
    org_id: str | None = None,
    from_id: int | None = None,
    limit: int = 100,
) -> list[Event]: ...

async def cost_summary(
    self, *,
    workgraph_id: str | None = None,
    agent_id: str | None = None,
) -> CostSummary: ...
```

`CostSummary` is a typed Pydantic dataclass:

```python
class CostSummary(BaseModel):
    total_cost_usd: float
    total_tokens: int
    total_turns: int
    task_count: int            # tasks contributing to the sum
    sample_event_count: int    # task.state_changed events seen
```

`PostgresEventRepository` implements the new methods with
parameterized SQL. `InMemoryEventRepository` implements them with
list comprehensions for the LSP-parametrized test pattern.

## D4 — Endpoint shapes (T2, T3, T4)

Following DESIGN §E. JSON response bodies are typed Pydantic
models declared in `src/vfobs/api/dto.py` so consumers can
generate clients. Schemas are versioned (`v: 1`).

### `GET /workgraphs/<id>` (T2)

```json
{
  "v": 1,
  "workgraph_id": "wg_x",
  "vtf": { "status": "doing", "kind": "infrastructure", "target_repos": ["..."], ... },
  "vfobs": {
    "event_count": 42,
    "last_event_id": 137,
    "last_event_type": "task.state_changed",
    "last_event_at": "2026-05-14T12:00:00Z"
  }
}
```

vtf metadata comes from the Adapter (T0). vfobs metadata comes
from T1's `find_by_workgraph(workgraph_id, limit=1, order=desc)`
plus a count.

### `GET /tasks/<id>` (T2)

Similar shape, with `vtf.task_status` + vfobs-derived last-event-of-task.

### `GET /tasks/<id>/events` (T2)

Paginated list:

```json
{
  "v": 1,
  "task_id": "t_x",
  "events": [ ...Event JSON... ],
  "next_from_id": 138
}
```

`next_from_id` is `events[-1].id + 1` (or null if fewer than `limit`
returned).

### `GET /events?filter=...&from=&limit=` (T3)

```json
{
  "v": 1,
  "events": [ ... ],
  "next_from_id": 138,
  "filter_applied": { "type": "task.heartbeat" }
}
```

Query params (Builder pattern in `_build_event_query(request) ->
EventQuery`):

- `workgraph_id`
- `task_id`
- `agent_id`
- `type` (full literal, e.g. `task.state_changed`)
- `type_namespace` (e.g. `task`)
- `org_id` (default `viloforge` extensibility hook)
- `from_id`
- `limit` (default 100, capped at 1000)

### `GET /workgraphs/<id>/cost` (T4)

```json
{
  "v": 1,
  "workgraph_id": "wg_x",
  "summary": {
    "total_cost_usd": 0.42,
    "total_tokens": 12345,
    "total_turns": 18,
    "task_count": 7,
    "sample_event_count": 7
  }
}
```

### `GET /agents/<id>/cost` (T4)

Same shape with `agent_id` instead of `workgraph_id`. Strategy
pattern on aggregation kind: `CostAggregationStrategy.by_workgraph(id)`
vs `CostAggregationStrategy.by_agent(id)` — both produce
`CostSummary`.

## D5 — Cost computation discipline

`cost_summary` aggregates over `task.state_changed` events whose
`data.execution_summary` is not null AND `data.to_status` is a
terminal status (`done`, `failed`, etc.). Logic in repository, not
in the endpoint layer, so the LSP test exercises it.

SQL approach (Postgres impl): `SELECT
sum((data->'execution_summary'->>'cost_usd')::numeric),
sum((data->'execution_summary'->>'total_tokens')::int), ...
FROM vfobs.events WHERE type = 'task.state_changed' AND
data ? 'execution_summary' AND (data->'execution_summary')
IS NOT NULL AND <workgraph_id|agent_id filter>`. NULL-tolerant —
missing execution_summary contributes 0.

In-memory: same logic in Python.

## D6 — Read auth applies to every WG2 endpoint

Every endpoint added by WG2 declares `Depends(get_read_principal)`
(the FastAPI dep that resolves `ReadAuth.verify(token)`). The
existing write endpoint POST /events still uses `Depends(get_principal)`
(ingest auth strategy). Two strategies coexist.

# Considered alternatives

## Alt A — Mirror vtaskforge's workgraph schema in vfobs

Rejected. Doubles the storage surface, creates a coherence problem
(which is the truth: vtaskforge or vfobs?). Adapter pattern is
correct.

## Alt B — Cost stored in a separate column / materialized view

Rejected for v1. Aggregations are cheap at v1's event volume
(<50M rows/month per partition). v2 can add a materialized view
if needed; the contract doesn't change.

## Alt C — Webhook from vtaskforge to vfobs for whoami push

Rejected. Adds operational complexity; pull-with-cache is simpler
and the cost is negligible.

# Open methodology questions (for retro)

- **MQ1.** The vtf-adapter's metadata projections (`WorkgraphMetadata`,
  `TaskMetadata`) are vfobs-side typed; if vtaskforge bumps its
  schema, our adapter silently drops new fields. Should the adapter
  surface "I dropped fields you might want"? Probably not for v1
  (operator visibility is fine via vtaskforge directly); revisit
  at v2.
- **MQ2.** The 60s whoami cache TTL is arbitrary; empirical
  validation in WG2 retro.
- **MQ3.** D5's "ignore tasks without execution_summary" silently
  excludes in-flight tasks from cost rollups. Document in the
  response body? Or let consumers infer from `task_count vs
  sample_event_count` discrepancy?

# References

- workgraph.md (this directory)
- DESIGN §E, §I; PLAN v0.2 §6, §15
- WG1 completion.md (what we're building on)
- `vtf-methodologies/spec-author/bugfix.md` R1-R14
