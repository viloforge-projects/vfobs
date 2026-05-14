---
id: t_v2scen
slug: scenario-read-side
title: Scenario test — read API + cost rollup end-to-end on kind
workgraph: read-api
order: 5
required_tags:
  - executor
depends_on:
  - t_v2cost
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-read-api/t5-scenario-read-side 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - "AC-T5-1 — tests/scenario/test_read_api_roundtrip.py runs the
    WG2 read paths end-to-end through the kind cluster (re-uses the
    WG1 scenario-prepare + a vtaskforge stub deployed alongside
    vfobs as a sidecar manifest)."
  - "AC-T5-2 — tests/fixtures/scenario-vtfstub.yaml deploys a tiny
    in-cluster vtaskforge stub (Python ASGI app in a single
    Deployment) that responds to /v2/auth/whoami, /v2/workgraphs/<id>/,
    /v2/tasks/<id>/. Implementation lives in
    scripts/scenario/vtfstub/{Dockerfile,app.py} (or use a single
    inline ConfigMap manifest if simpler — pick what's cleanest)."
  - "AC-T5-3 — values-scenario.yaml extends to set VFOBS_VTASKFORGE_URL
    to the in-cluster vtfstub service URL."
  - "AC-T5-4 — Scenario test 'test_full_read_path_after_seed':
    POSTs ~10 events spanning 2 workgraphs + 2 agents with varied
    execution_summary, then GETs each endpoint, asserts response
    payloads + cost math."
  - "AC-T5-5 — Scenario test 'test_read_endpoints_require_auth':
    unauthenticated GET to each read endpoint returns 401."
  - "AC-T5-6 — Scenario test 'test_filter_pagination_e2e':
    POSTs 250 events, GETs /events?limit=100, asserts 100 returned
    with next_from_id; follows the cursor to page 2 + 3."
  - "AC-T5-7 — Updated docs/scenario-test-runbook.md describes the
    new vtfstub sidecar and how to operate WG2 scenario tests."
  - "AC-T5-8 — Branch pushed."

---

# Spec

## Files touched

- `tests/scenario/test_read_api_roundtrip.py` (new)
- `tests/fixtures/scenario-vtfstub.yaml` (new)
- `scripts/scenario/vtfstub/Dockerfile` (new — or skip + use a
  busybox+python -m http.server with a python script; cleaner is a
  real Dockerfile)
- `scripts/scenario/vtfstub/app.py` (new — ASGI app)
- `scripts/scenario/prepare.sh` (modify — build + load + apply
  vtfstub; update vfobs helm values with the stub URL)
- `tests/fixtures/values-scenario.yaml` (modify — vtaskforge URL)
- `docs/scenario-test-runbook.md` (modify)

## Implementation sketch

**No new patterns — scenario test composes WG2 endpoints and a
trivial vtaskforge stub.**

### vtfstub/app.py shape

```python
from fastapi import FastAPI, Header
app = FastAPI()

@app.get("/v2/auth/whoami")
def whoami(authorization: str = Header(...)):
    # any non-empty token is valid for the stub
    return {"user_id": "stub-operator", "display_name": "Stub Operator"}

@app.get("/v2/workgraphs/{workgraph_id}/")
def get_workgraph(workgraph_id: str):
    return {"id": workgraph_id, "status": "doing", "kind": "infrastructure",
            "target_repos": ["viloforge/vfobs"], "tags": []}

@app.get("/v2/tasks/{task_id}/")
def get_task(task_id: str):
    return {"id": task_id, "workgraph_id": "wg_scenario_read",
            "status": "doing", "title": "Stub Task"}
```

This stub exists ONLY so VtfClient has a real network target in the
scenario; production-mode read auth + adapter behavior is verified
in unit + integration tests against MockTransport. The scenario
verifies "the deployed shape works", not "vfobs + real vtaskforge
agree" — that's a higher-level integration that lands once we have
a real vtf-canary scenario in WG5+.

## Fail-loud directive (R3)

If any required step (vtfstub image build, kind load, vfobs+stub
helm-install ordering, branch push) cannot be completed: report
the failure explicitly. **In particular, do NOT mark this task
complete if the scenario tests do not run to green end-to-end.**
The WG1 T6 fail-loud precedent applies.

# Constraints

- vtfstub MUST be tiny + dependency-free (FastAPI + uvicorn only).
- Scenario test re-uses the WG1 kind cluster setup; the only delta
  is the additional vtfstub Deployment + a Settings env var.
- Tests MUST exercise auth on every read endpoint (401 path).

# Out of scope

- Realistic vtaskforge fixture (a true vtaskforge deployment in
  kind) — deferred. The stub is enough for scenario contract.
- Cost-anomaly scenario (v2).
- Multi-cluster scenarios.

# References

- plan.md §D1-§D5
- WG1 T6 scenario-roundtrip — pattern to extend
- `docs/scenario-test-runbook.md`
