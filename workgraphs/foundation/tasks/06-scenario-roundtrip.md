---
id: t_A5dP7s
slug: scenario-roundtrip
title: Scenario test — vfobs+Postgres in kind, event roundtrip end-to-end
workgraph: foundation
order: 6
required_tags:
  - executor
depends_on:
  - t_Z9bN2m
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t6-scenario-roundtrip 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T6-1 — `Makefile` gains target `test-scenario` (extending or
    replacing the WG0 placeholder) that runs
    `pytest -m scenario tests/scenario/`. A second target
    `scenario-prepare` runs the kind bootstrap (cluster create,
    Postgres deploy, vfobs build+load+deploy). `test-scenario`
    depends on `scenario-prepare` unless `SKIP_PREPARE=1`.
    A third target `scenario-teardown` deletes the kind cluster.
  - AC-T6-2 — `scripts/scenario/prepare.sh` (new) script:
    (a) verifies `kind`, `helm`, `kubectl`, `docker` are installed
        (fails fast with install hints if not)
    (b) `kind create cluster --name vfobs-scenario` (idempotent —
        reuses if exists when `KEEP_CLUSTER=1`)
    (c) deploys a minimal Postgres via
        `helm upgrade --install vfobs-pg bitnami/postgresql
        --version <pinned>` into namespace `vfobs-test`, with a
        known root password and a `vfobs_app` user pre-created
    (d) bootstraps the vfobs schema by running
        `alembic upgrade head` against that Postgres (via a
        port-forward and a local Python invocation)
    (e) `docker build -t viloforge/vfobs:scenario .`
    (f) `kind load docker-image viloforge/vfobs:scenario --name vfobs-scenario`
    (g) deploys vfobs via `helm install vfobs ./charts/vfobs
        --values tests/fixtures/values-scenario.yaml --namespace
        vfobs-test`, overriding image.tag, the DB DSN, and a known
        test ingest token. ESO is disabled in scenario values; the
        scenario values include a literal Secret manifest instead.
    (h) waits for vfobs pod readiness (kubectl wait
        --for=condition=ready pod -l app=vfobs -n vfobs-test
        --timeout=120s)
  - AC-T6-3 — `Dockerfile` at repo root (new) — multi-stage build:
    builder installs the package in editable form, final stage is
    a slim runtime image starting `uvicorn vfobs.main:create_app
    --factory --host 0.0.0.0 --port 8080`. Health probes work
    against the built image (validated by T6's scenario, not a
    separate unit).
  - AC-T6-4 — `tests/scenario/test_event_roundtrip.py` test
    `test_post_event_persists_and_retrievable` (scenario marker):
    (a) opens `kubectl port-forward svc/vfobs 18080:8080
        -n vfobs-test` (subprocess managed by a pytest fixture)
    (b) opens `kubectl port-forward svc/vfobs-pg-postgresql
        15432:5432 -n vfobs-test` (subprocess fixture)
    (c) POSTs a real `TaskStateChanged` event to
        `http://localhost:18080/events` with the test ingest token
        in the Authorization header
    (d) asserts 201 + id is a positive integer
    (e) connects to `localhost:15432` as `vfobs_app`, queries
        `SELECT * FROM vfobs.events WHERE id = <returned id>`
    (f) asserts every base field matches what was POSTed, `data`
        JSONB matches deep-equal, and `server_received_at` was
        injected by the Enricher (visible in the JSONB data)
  - AC-T6-5 — `tests/scenario/test_event_roundtrip.py` test
    `test_health_endpoints_reachable_through_chart` (scenario
    marker): GETs `/healthz` and `/readyz` through the port-forward,
    asserts 200 + expected body shapes (verifies T3 endpoints
    survived through the Helm chart deployment).
  - AC-T6-6 — `tests/scenario/test_event_roundtrip.py` test
    `test_unauthorized_post_rejected` (scenario marker): POSTs an
    event with no Authorization header → 401; POSTs with wrong
    token → 401. Verifies T4 auth survived through the chart.
  - AC-T6-7 — Documentation in
    `docs/scenario-test-runbook.md` (new) explains how an operator
    runs the scenario test: `make scenario-prepare`,
    `make test-scenario`, `make scenario-teardown`. Includes
    `KEEP_CLUSTER=1` for fast iteration, troubleshooting tips,
    and a note that the scenario test is **not** wired to CI in
    WG1 (manual operator run) per the plan's CI deferral.
  - AC-T6-8 — Branch `wg-vfobs-foundation/t6-scenario-roundtrip` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `Makefile` (modify) — add `test-scenario`, `scenario-prepare`,
  `scenario-teardown` targets. Existing WG0 placeholder for
  `test-scenario` is replaced.
- `Dockerfile` (new) — multi-stage; runtime image hosts the
  FastAPI app via uvicorn.
- `scripts/scenario/__init__.py` (new — empty, package marker).
- `scripts/scenario/prepare.sh` (new — bash; orchestrates kind
  bootstrap).
- `scripts/scenario/teardown.sh` (new — bash; `kind delete
  cluster --name vfobs-scenario`).
- `tests/scenario/test_event_roundtrip.py` (new).
- `tests/scenario/conftest.py` (new) — `port_forward` pytest
  fixture (subprocess.Popen with cleanup).
- `tests/fixtures/values-scenario.yaml` (new) — Helm values for
  the scenario environment (ESO disabled, literal secret values,
  image.tag=scenario, replicaCount=1, monitoring.enabled=false).
- `tests/fixtures/scenario-secrets.yaml` (new) — literal k8s
  Secret manifest applied alongside the chart so the deployment's
  secretKeyRefs resolve without ESO.
- `docs/scenario-test-runbook.md` (new) — operator runbook.

(NOT touched: anything under `src/vfobs/` — T6 exercises the
service as-built, not modifying it. If the scenario test reveals a
defect, that's a bug to fix in T0-T4, not patch in T6.)

## Implementation sketch

**Patterns: n/a — test/infra task. The scenario test exercises the
patterns deployed by T0-T5 in their integrated form.**

The single scenario test is the closest WG1 gets to "is the system
actually wired up and working?" Its purpose is to catch failure
modes that unit + integration cannot:

- Chart-template defects that only surface at deploy
- Probe-path mismatches between T3 and T5
- DSN encoding bugs that only matter when the connection-string
  passes through Helm + Secret + env-var resolution
- Image-build defects (missing files, wrong entrypoint)
- DB-grants gotchas (T0 ACs assert grants on a temp DB; the
  scenario assert the same grants resolve correctly in the
  scenario DB)

### Makefile excerpt

```make
KIND_CLUSTER ?= vfobs-scenario
KEEP_CLUSTER ?= 0

scenario-prepare:
	bash scripts/scenario/prepare.sh

test-scenario:
ifeq ($(SKIP_PREPARE),1)
	pytest -m scenario tests/scenario/
else
	$(MAKE) scenario-prepare
	pytest -m scenario tests/scenario/
endif

scenario-teardown:
	bash scripts/scenario/teardown.sh
```

### scripts/scenario/prepare.sh shape

```bash
#!/usr/bin/env bash
set -euo pipefail

for tool in kind helm kubectl docker; do
  command -v "$tool" >/dev/null || { echo "Missing: $tool"; exit 1; }
done

CLUSTER="${KIND_CLUSTER:-vfobs-scenario}"
NS="vfobs-test"

if ! kind get clusters | grep -qx "$CLUSTER"; then
  kind create cluster --name "$CLUSTER"
fi

kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -

helm repo add bitnami https://charts.bitnami.com/bitnami 2>/dev/null || true
helm upgrade --install vfobs-pg bitnami/postgresql \
  --version 15.5.20 \
  --namespace "$NS" \
  --set auth.username=vfobs_app \
  --set auth.password=devpassword \
  --set auth.database=vfobs \
  --wait --timeout 5m

# bootstrap schema
kubectl port-forward svc/vfobs-pg-postgresql 15432:5432 -n "$NS" &
PF_PID=$!
trap "kill $PF_PID 2>/dev/null || true" EXIT
sleep 3
VFOBS_DATABASE_URL="postgresql+asyncpg://vfobs_app:devpassword@localhost:15432/vfobs" \
  alembic upgrade head
kill $PF_PID
trap - EXIT

docker build -t viloforge/vfobs:scenario .
kind load docker-image viloforge/vfobs:scenario --name "$CLUSTER"

kubectl apply -f tests/fixtures/scenario-secrets.yaml -n "$NS"
helm upgrade --install vfobs ./charts/vfobs \
  --values tests/fixtures/values-scenario.yaml \
  --namespace "$NS" \
  --wait --timeout 5m

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vfobs \
  -n "$NS" --timeout=120s
```

### test_event_roundtrip.py shape

```python
import pytest, httpx, psycopg, json
from datetime import datetime, UTC

@pytest.mark.scenario
async def test_post_event_persists_and_retrievable(vfobs_port, pg_port):
    event = {
        "type": "task.state_changed",
        "v": 1,
        "workgraph_id": "wg_scenario1",
        "task_id": "t_scenario",
        "source": "scenario-test",
        "timestamp": datetime.now(UTC).isoformat(),
        "data": {"from_status": "todo", "to_status": "doing"},
    }
    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"http://localhost:{vfobs_port}/events",
            json=event,
            headers={"Authorization": "Bearer scenario-test-token"},
        )
    assert resp.status_code == 201
    eid = resp.json()["id"]
    assert isinstance(eid, int) and eid > 0

    with psycopg.connect(
        f"host=localhost port={pg_port} user=vfobs_app password=devpassword dbname=vfobs"
    ) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT type, workgraph_id, task_id, data FROM vfobs.events WHERE id = %s", (eid,))
            row = cur.fetchone()
    assert row is not None
    typ, wgid, tid, data = row
    assert typ == "task.state_changed"
    assert wgid == "wg_scenario1"
    assert tid == "t_scenario"
    assert data["from_status"] == "todo"
    assert data["to_status"] == "doing"
    assert "server_received_at" in data  # Enricher fingerprint
```

The `vfobs_port` / `pg_port` fixtures in `conftest.py` start
`kubectl port-forward` subprocesses, sleep until the port is
reachable, yield the local port, and `kill` the subprocess on
teardown.

## Fail-loud directive (R3)

If any required step (kind/helm/docker missing, image build
failure, deployment timeout, scenario script bug, branch push)
cannot be completed due to missing dependencies, missing tools, or
blocked external constraints: report the failure explicitly in
your completion notes — do not rationalize partial completion as
success. **In particular, do NOT mark this task complete if the
scenario test does not actually run to green end-to-end** —
ghost-completion at the scenario level is the worst class of
failure for an observability service. The judge will fail tasks
that report dishonest success; tasks that report failure honestly
can be reworked.

# Constraints

- The scenario test MUST run end-to-end on the operator's
  workstation today. It MUST NOT depend on CI to work.
- Scenario teardown MUST be idempotent — running
  `scenario-teardown` against a non-existent cluster MUST NOT
  fail the operator session.
- The scenario test MUST use real network (port-forward), not
  in-process clients. The whole point is to exercise the deployed
  shape.
- The bitnami/postgresql chart version MUST be pinned (no `latest`).
- Tokens / passwords used in the scenario are dev-only literals
  (acceptable per `feedback_dev_convenience_password` discipline)
  but MUST NOT leak into the shared Vault path or any real Secret
  applied to production clusters.

# Out of scope

- Scenario tests for read endpoints (WG2 work).
- Scenario tests for SSE (WG3).
- Scenario tests for anomaly detection (WG4).
- Wiring the scenario test into CI. Per IMPLEMENTATION-PLAN §13,
  CI is a follow-up `kind: infrastructure` workgraph.
- Multi-replica / HA scenarios.
- Cross-cluster / multi-region.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §F, NFR4
- `vtf-methodologies/spec-author/bugfix.md` R10 (scenario test
  delivery mechanism: Makefile target today, CI later)
- Reference scenario: `viloforge-projects/vafi/workgraphs/vafi-rolling-restart-fix/tasks/03-integration-test-rolling-restart.md`
  — same shape (Makefile-driven kind + pytest scenario)
- Bitnami PostgreSQL chart:
  https://github.com/bitnami/charts/tree/main/bitnami/postgresql
