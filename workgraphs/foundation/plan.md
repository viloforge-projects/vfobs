---
authored_by: claude-opus-4-7
co_authors:
  - operator:lonvilo
created_at: 2026-05-14T00:00:00Z
last_revision: 2026-05-14T00:00:00Z
---

# Approach

Build vfobs's foundation as seven small tasks that compose into "an
HTTP service that accepts typed observability events and persists
them durably." Strict topological order through the DAG (workgraph.md),
parallel where possible.

This plan also resolves two of the IMPLEMENTATION-PLAN's deferred
open questions:

- **OIQ4 — event ID scheme.** Resolved to **Postgres `BIGSERIAL`
  (bigint sequence)**. Globally monotonic within a single Postgres
  instance, perfect for SSE `Last-Event-ID` (just emit `id` as the
  SSE id), zero new dependencies, sorts numerically. ULID was
  considered for cross-instance ordering but is YAGNI: v1 is a
  single instance and even at the v2 multi-instance horizon the
  Postgres `id` continues to work for in-instance ordering — clients
  resuming a stream from a specific instance use that instance's
  ids. The cost of switching later is one migration. Locked.
- **OIQ3 (write half) — write API auth.** Resolved to **shared
  service-account token** in a Kubernetes Secret (ESO-managed from
  Vault), validated by a `StaticTokenAuth` Strategy. Future iterations
  can introduce per-pod tokens or mTLS by swapping the Strategy
  implementation — the `IngestAuth` interface admits both without
  endpoint changes. The DESIGN doc's §I "vtf-token piggyback auth"
  language applies to the **read** API (WG2), not the write API;
  controllers are first-class internal clients with their own
  credentials per §F. The IMPLEMENTATION-PLAN's OIQ3 framing
  conflated read and write; this plan splits them cleanly.

OIQ3's read-half (operator CLI / web UI auth) remains deferred to
WG2.

# Design decisions (locked for WG1)

## D1 — Single `events` table, partitioned monthly by `timestamp`

One Postgres table, `vfobs.events`, partitioned by month on the
`timestamp` column (event time, what queries care about). Partition
key must be part of the primary key in Postgres, so PK = `(id,
timestamp)`. `id` is independently unique via the `BIGSERIAL`
sequence; the composite PK is a Postgres mechanical requirement, not
a semantic statement.

Why monthly granularity:

- Retention/archival happens by dropping old partitions — trivially
  fast vs row-level DELETE.
- Single-month partitions stay small enough to index efficiently
  (estimate: <50M rows/month at peak v1 load; well within Postgres
  comfort).
- A future archival worker can move old partitions to cheaper
  storage (e.g., S3 via foreign-data-wrapper) without touching
  application code.

Why one table (not per-namespace tables):

- Schema versioning lives on the row (`v` column) not on the table,
  per NFR1. Per-namespace tables couple schema evolution to physical
  layout.
- Cross-namespace queries (e.g., "all events for workgraph W") are
  index scans on `(workgraph_id, id)` without union-all.
- The `type` column already encodes the namespace; we keep
  `event_type_check` as a CHECK constraint that enforces the
  namespace prefix.

## D2 — Event base columns (locked v1 schema)

The `vfobs.events` table columns:

| Column | Type | Constraints | Source |
|---|---|---|---|
| `id` | `BIGSERIAL` | NOT NULL | Server-assigned monotonic |
| `v` | `SMALLINT` | NOT NULL DEFAULT 1 | Schema version per NFR1 |
| `workgraph_id` | `TEXT` | NOT NULL | DESIGN §C (G9 required) |
| `task_id` | `TEXT` | NULL | DESIGN §C (most events) |
| `agent_id` | `TEXT` | NULL | DESIGN §C |
| `trace_id` | `TEXT` | NULL | DESIGN §C |
| `source` | `TEXT` | NOT NULL | DESIGN §C |
| `type` | `TEXT` | NOT NULL CHECK matches `^(workgraph|task|harness|gate|judge|anomaly)\.` | DESIGN §C |
| `timestamp` | `TIMESTAMPTZ` | NOT NULL | Event-time (client-supplied) |
| `data` | `JSONB` | NOT NULL DEFAULT `'{}'::jsonb` | Typed payload |
| `classification` | `TEXT` | NOT NULL DEFAULT `'internal'` CHECK in (`'public'`,`'internal'`,`'secret'`) | NFR3 |
| `org_id` | `TEXT` | NOT NULL DEFAULT `'viloforge'` | NorthStar extensibility |
| `cluster_id` | `TEXT` | NOT NULL DEFAULT `'vafi-dev'` | NorthStar extensibility |
| `created_at` | `TIMESTAMPTZ` | NOT NULL DEFAULT `now()` | Server-time audit |

Primary key: `(id, timestamp)` (partition-key requirement).

Indexes:
- `events_workgraph_id_id_idx` on `(workgraph_id, id)` — main
  filter-by-workgraph query
- `events_task_id_id_idx` on `(task_id, id)` WHERE `task_id IS NOT NULL`
  — filter-by-task
- `events_type_id_idx` on `(type, id)` — filter-by-event-type
- `events_timestamp_idx` on `(timestamp)` — range scans for
  time-window queries
- `events_org_cluster_idx` on `(org_id, cluster_id, id)` — future
  multi-tenant filtering (deferred but extensibility hook is here)

## D3 — Event types module shape (T1)

`src/vfobs/events/`:

```
events/
├── __init__.py            # public exports (Event, factory, namespaces enum)
├── base.py                # Event base dataclass + EventNamespace enum + Classification enum
├── factory.py             # EventFactory: typed constructors (Factory pattern)
├── namespaces/            # one module per namespace; events as data
│   ├── __init__.py
│   ├── workgraph.py       # workgraph.created, .state_changed, .completed
│   ├── task.py            # task.claimed, .state_changed, .heartbeat, .workdir_changed
│   ├── harness.py         # harness.turn_started, .tool_call, .turn_completed
│   ├── gate.py            # gate.started, .passed, .failed
│   ├── judge.py           # judge.started, .review_submitted, .decision
│   └── anomaly.py         # anomaly.stuck_detected (v1 only)
└── schema.py              # JSON Schema export for contract testing
```

Each event is a `pydantic.BaseModel` subclass with:
- `v: int = 1`
- `type: Literal["<namespace>.<verb>"]`
- `workgraph_id: str`
- `task_id: str | None = None`
- `agent_id: str | None = None`
- `trace_id: str | None = None`
- `source: str`
- `timestamp: datetime`
- `classification: Classification = Classification.INTERNAL`
- `org_id: str = "viloforge"`
- `cluster_id: str = "vafi-dev"`
- `data: <typed sub-model>` per event type

The `data` sub-model is what makes each event type distinct — e.g.,
`TaskStateChangedData(from_status: str, to_status: str,
execution_summary: ExecutionSummary | None)`.

Pattern: **Factory** (`EventFactory.task_state_changed(...) ->
TaskStateChanged`) keeps event construction in one place; consumers
import `EventFactory`, not individual event classes. **Specification**
(events-as-data) keeps event types declarative — no business logic
on events themselves; behavior lives in repositories and handlers.

## D4 — EventRepository interface + Postgres impl (T2)

`src/vfobs/repositories/event_repository.py`:

```python
class EventRepository(ABC):
    @abstractmethod
    async def store(self, event: Event) -> int:
        """Persist event, return the assigned id."""

    @abstractmethod
    async def get_by_id(self, event_id: int) -> Event | None:
        """Retrieve a single event by id, for testing/debugging."""
```

Two implementations:
- `PostgresEventRepository` — production impl, SQLAlchemy + asyncpg.
  Inserts via parametrized query; reads back the assigned `id` via
  `RETURNING id`.
- `InMemoryEventRepository` — test double for unit tests in T2 and
  downstream. Backed by a plain list; preserves insertion order.

The interface is async-only — controllers calling vfobs are async
(matches vtaskforge SDK pattern); making the repo sync would force
a sync↔async bridge in the API layer.

## D5 — FastAPI app structure (T3)

`src/vfobs/`:

```
main.py                    # FastAPI app factory, lifespan, route mounting
config.py                  # Settings (pydantic-settings BaseSettings) — Singleton
db.py                      # SQLAlchemy async engine + session factory
api/
  __init__.py
  health.py                # /healthz, /readyz
  metrics.py               # /metrics (prometheus_client)
  events.py                # POST /events (T4)
  auth.py                  # IngestAuth strategy + FastAPI dependency
middleware/
  __init__.py
  logging.py               # request-id + structured log Decorator
```

Patterns: **Singleton** for `Settings` (via `pydantic-settings`, with
a module-level `get_settings()` LRU-cached factory). **Decorator** for
request logging (FastAPI middleware wraps each request, attaches a
`request_id` to the log context).

Health endpoints:
- `/healthz` — liveness; returns `{"status": "ok"}` always (process alive)
- `/readyz` — readiness; returns `{"status": "ok", "db": "ok"}` on
  successful trivial DB query, 503 + detail otherwise (gates k8s
  traffic until DB is reachable)
- `/metrics` — Prometheus exposition (request counter + histogram via
  `prometheus_client`). Standalone counter for events ingested.

## D6 — Write API: POST /events (T4)

`POST /events` with:
- **Auth dependency** — `IngestAuth.verify(token)` via FastAPI's
  `Depends`. The Strategy `StaticTokenAuth` reads the token from
  `Settings.ingest_token` (env-injected from a k8s Secret). Wrong
  token → 401. Missing token → 401.
- **Body** — JSON. Validates against the union of all event types
  defined in T1 (FastAPI auto-derives validation from the Pydantic
  models). Bad shape → 422.
- **Chain of Responsibility** internal pipeline:
  1. **Validator** — Pydantic validation (auto, before our code runs)
  2. **Enricher** — fills server-derived fields (`server_received_at`
     stored in `data` if not present; resolves `source` from the
     authenticated principal if absent)
  3. **Storer** — calls `EventRepository.store(event)`
- **Response** — `201 Created` with `{"id": <int>}` (the assigned id).

Rate-limiting, batching, and replay protection are explicit
non-goals for v1 (see workgraph.md "Out of scope"). A single
controller pushes events at well below Postgres ingest capacity for
v1's scale; per DESIGN §D, hardening waits for production-scale
needs.

## D7 — Helm chart layout (T5)

`charts/vfobs/` with conventional layout:

```
Chart.yaml
values.yaml                # service config: image, replicas, resources, env, dbConn
templates/
  _helpers.tpl
  deployment.yaml          # one replica in v1 (single-tenant)
  service.yaml             # ClusterIP, port 8080
  serviceaccount.yaml
  configmap.yaml           # non-secret settings
  externalsecret.yaml      # ESO ExternalSecret pulling INGEST_TOKEN + DB password from Vault
  servicemonitor.yaml      # Prometheus scrape (optional, gated by .Values.monitoring.enabled)
  poddisruptionbudget.yaml # minAvailable=0 in v1 (single replica)
NOTES.txt
```

The chart references the image as `viloforge/vfobs:{{ .Chart.AppVersion }}`
and admits override via `image.tag`. ArgoCD manifest in
`viloforge-platform/argo/products/vfobs/` (out of scope here; lands
as an ops follow-up) references this chart path.

Build of `viloforge/vfobs` Docker image is also out of scope for
WG1 — the chart works against any image tag. The image-build CI
pipeline is a follow-up infrastructure workgraph (matches the
plan's §13 deferral of CI).

## D8 — Scenario test infrastructure (T6)

`tests/scenario/test_event_roundtrip.py` runs via `make
test-scenario`. The Makefile target:

1. Verifies `kind` and `helm` are installed (fails fast with hints).
2. `kind create cluster --name vfobs-scenario` (or reuses if
   `KEEP_CLUSTER=1`).
3. Deploys a minimal Postgres (bitnami/postgresql or similar) via
   helm to namespace `vfobs-test`.
4. Bootstraps the vfobs schema by running the T0 migration against
   that Postgres.
5. Loads the locally-built vfobs image into kind via `kind load
   docker-image`.
6. Deploys vfobs via the T5 chart (overriding `dbConn` to the local
   Postgres + `INGEST_TOKEN` to a known test value).
7. Runs `pytest -m scenario tests/scenario/test_event_roundtrip.py`.
8. Tears down kind (unless `KEEP_CLUSTER=1`).

The pytest test:
- POSTs a `task.heartbeat` event to `vfobs.vfobs-test.svc.cluster.local:8080/events`
  via a `kubectl port-forward` (or via a small in-cluster job pod).
- Reads back from Postgres directly (psycopg in the test, connecting
  to the same port-forwarded DB).
- Asserts the round-tripped event equals the POSTed event (minus
  server-assigned fields).

The scenario explicitly verifies what unit + integration cannot:
that **a real ingress, a real network, a real Postgres, and a real
k8s deployment together produce the expected end-to-end behavior**.

# Considered alternatives

## Alt A — Use ULID for event IDs instead of BIGSERIAL

Rejected. ULID is lexicographically sortable and works across
instances, but v1 is a single Postgres instance — there's no
correctness benefit. ULID adds a dependency and a 26-character
string id that's larger than an 8-byte bigint. The migration cost
if we ever need it is trivial (one column, one index swap). Keep
BIGSERIAL.

## Alt B — Per-namespace tables (events_workgraph, events_task, ...)

Rejected. Coupling schema evolution to physical table layout (NFR1
violation) and forcing union-all queries for cross-namespace reads
(WG2 + WG3 hot path) is wrong. The single-table-with-type-column
shape is canonical for event-log designs.

## Alt C — Synchronous repository interface

Rejected. The whole call chain (FastAPI handler → repository →
asyncpg) is async-native; introducing a sync repo would force a
`run_in_executor` bridge that buys nothing.

## Alt D — Skip the scenario test in WG1; defer to WG5

Rejected. The whole point of the scenario level is to catch the
"works on my machine, broken in production" defects that integration
tests miss. Foundational deployment shape (chart + Postgres + write
API) must be exercised end-to-end before WG2-5 build on it. Deferring
the scenario test compounds the risk that a chart bug surfaces only
in WG5.

## Alt E — Use Alembic vs raw SQL migration

**Accepted: Alembic.** Already in the WG0 `pyproject.toml`
dependencies. Alembic gives migration tracking (the `alembic_version`
table tells you what schema state a DB is in) which is necessary
the moment a second migration lands. Raw SQL would need us to
reinvent that.

# Open methodology questions (for retro)

- **MQ1.** The IMPLEMENTATION-PLAN says T0's gate verifies the
  external Postgres deployment; this plan defers actual deploy to
  T5 + T6, so T0's external artifact is just "branch pushed +
  migration file present + migration runs cleanly against a temp
  DB in pytest." Is that enough R1 strength? Empirical answer
  pending the verifier stage.
- **MQ2.** T5 (Helm chart) doesn't deploy to a real cluster either
  — only `helm template` + `helm lint`. The actual deploy happens
  in T6's scenario test. Is "chart renders" a strong enough R1 AC
  for T5, or should T5 also assert the chart was pushed to a
  registry? v1 has no chart registry yet; deferred.
- **MQ3.** The "operator implements" mode (PLAN §11) means I run
  the test_command gates manually before transitioning vtaskforge
  tasks. When vafi-executor becomes reliable and these are
  re-routed, the gate definitions must remain mechanically runnable
  with no operator-specific shims. The current gates are pure shell
  (git ls-remote + pytest) — should pass that test.

# References

- workgraph.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §C, §D, §F, §I
- `viloforge-platform/docs/pipeline-observability-IMPLEMENTATION-PLAN.md` §5, §15
- `viloforge-platform/docs/engineering-principles.md` §3, §5
- `vtf-methodologies/spec-author/bugfix.md` R1-R12
- `viloforge/vfobs/CLAUDE.md` (repo-local discipline reminder)
- `viloforge/vfobs/pyproject.toml` (dependencies locked at WG0)
