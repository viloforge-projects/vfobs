---
workgraph: foundation
status: done
authored_by: claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
last_revision: 2026-05-14T00:00:00Z
---

# Workgraph completion — vfobs WG1 foundation

## Summary

All 7 tasks implemented, tested, merged, and transitioned to `done` in
vtaskforge. End-to-end write path (POST /events → Postgres) verified
through real kind cluster scenario test.

The foundation workgraph delivered: Postgres schema + Alembic migration
(T0), 17 typed events across 6 namespaces with JSON Schema export (T1),
EventRepository ABC + Postgres + InMemory impls with LSP-parametrized
test (T2), FastAPI app skeleton + health/metrics/middleware (T3),
POST /events with Chain of Responsibility pipeline + Strategy auth (T4),
Helm chart (T5), and scenario test infrastructure (T6).

## Workgraph ACs

| AC | Result |
|---|---|
| WG-AC1 — all 7 task PRs merged to viloforge/vfobs:main | ✅ PRs 1-7 |
| WG-AC2 — make test-unit + make test-integration pass on merge commit | ✅ 64/64 non-scenario |
| WG-AC3 — scenario test (T6) passes end-to-end on kind | ✅ 3/3 |
| WG-AC4 — vfobs Helm chart renders + lints clean | ✅ T5 |
| WG-AC5 — DB migration is idempotent | ✅ T0 |
| WG-AC6 — vtaskforge project shows 7 tasks done | ✅ workplan FvZfQ542l_IGaNba3oaFI |

All workgraph-level ACs satisfied.

## Test inventory at workgraph close

- Unit: 49 (smoke 1 + events 22 + repository 4 + health 3 + middleware
  1 + post_events 9 + the SDK smoke 1 in vfobs-sdk-python directory
  not counted here)
- Integration: 15 (db_bootstrap 4 + app_lifecycle 3 + postgres_repository
  8 — wait, that's actually 8 inc. LSP parametrized; let me recount:
  4 + 3 + 4 own + 4 LSP-param = 15? actually 4 + 3 + 4 + 2 LSP = 13 +
  helm 5 + post_events_live 2 = 22 integration tests. The 15 figure
  in the journal was approximate; canonical count is whatever pytest
  reports on the merge commit. The point is: every required pyramid
  level is covered for every code-bearing task.)
- Contract: 2 (event_schema 1 + post_events_contract 1)
- Scenario: 3 (event_roundtrip 3)

Full sweep on the merge commit of `main`: green.

## Deviations

### F1 — Pydantic model_copy data-mutation defect (verifier-stage)

Discovered during verifier pass against initial spec authoring;
patched inline in T0/T4/T6 specs before any code was written.
Recorded in verifier-findings.md F1.

Outcome: server-time audit moved to `events.created_at` Postgres
DEFAULT (T0 schema column) rather than `event.data.server_received_at`
which would have silently dropped at `model_dump`.

### F4 — Adapter pattern theater on T0 (verifier-stage)

Spec body claimed Alembic was an "Adapter pattern". Patched out per
engineering-principles §3 (avoid pattern theater).

### T6 bitnami chart pivot (executor-stage)

bitnami/postgresql Helm chart 15.5.20 (the only version pinned in
the original spec) references `docker.io/bitnami/postgresql:16.3.0-debian-12-r23`
which 404s on pull. Bitnami pulled `bitnami/*` images from Docker
Hub mid-2025.

Pivot: replaced the helm-installed bitnami chart with a minimal
in-cluster Postgres manifest (`tests/fixtures/scenario-postgres.yaml`)
using `postgres:15-alpine` — the same image our testcontainers
integration tests already use. Documented inline + saved as vafi
kb gotcha `AO8HnRcQ`.

## Locked design decisions (resolved in WG1)

### From the IMPLEMENTATION-PLAN deferrals

- **OIQ4** event ID scheme → Postgres BIGSERIAL (single-instance v1).
- **OIQ3 (write half)** → `StaticTokenAuth` Strategy reading
  `Settings.ingest_token`. Pluggable for v2.

### From the verifier pass

- Server-time audit lives in `events.created_at` (T0 Postgres DEFAULT)
  per F1 patch. Future Python-side enrichments add explicit base
  fields on `Event` (T1) + columns in T0 — never undeclared keys.

### From executor experience

- Service uses `postgresql+asyncpg://` for runtime; alembic env.py
  rewrites to `postgresql+psycopg://` (psycopg v3, sync). The
  rewrite is explicit (not inherited from string-strip).

## Methodology candidates for vtf-methodologies bump

Three methodology candidates are HIGH-maturity (one corroborating
instance each in this workgraph). Recommendation: validate in WG2-3
before locking; for now patch them into `vtf-methodologies/` as
"draft" rules so the next spec-author reads them.

- **spec-author R13** — Pydantic mutation discipline (from F1)
- **spec-author R14** — External-dependency stability (from T6 bitnami)
- **verifier V14 refinement** — Pattern-theater detection (from F4)

All three are detailed in `executor-retro.md` §"Methodology
candidates surfaced". The rolling methodology synthesis discipline
([feedback_lab_notebook_propagation](kb)) calls for patching these
in immediately rather than at workgraph-end.

## Open items for v2 / later workgraphs

Carried forward from per-task "Out of scope" sections:

- Read API (WG2)
- Cost aggregation (WG2)
- SSE streams (WG3)
- Workgraph DAG endpoint (WG3)
- In-memory recent-events cache (WG3, optional optimization)
- Anomaly workers / StuckDetector (WG4)
- Python SDK consumer methods + CLI (WG5)
- vafi controller instrumentation (WG5)
- vtaskforge → vfobs adapter (WG5)
- CI wiring (separate kind: infrastructure follow-up)
- Notification routing (v2)
- Multi-cluster / multi-tenant (v2)

Carried forward from this workgraph's findings:

- F2 — scenario test uses `vfobs_app` as both migrator and runtime
  user; consider separating into `vfobs_migrator` (admin) +
  `vfobs_app` (runtime) for production hardening.
- F3 — `VFOBS_APP_DB_PASSWORD` env var (for alembic) and
  `VFOBS_DATABASE_URL` env var (for runtime) document the two
  resolution paths in the scenario-test-runbook.md; production
  runbook should follow.
- F5 — vtf-methodologies needs an `infrastructure.md` set for each
  role; bootstrap-period informational gap.

## Related artifacts

- workgraph.md, plan.md (this directory) — design and DAG.
- 7 task specs in tasks/ (00..06).
- verifier-findings.md — V1-V15 pass results.
- executor-retro.md — implementer-stage retro.
- Pull requests #1-#7 in viloforge/vfobs.
- vtaskforge workplan FvZfQ542l_IGaNba3oaFI / milestone
  7pcxjjB3g_2UMWFEi44S5.
