---
id: t_Z9bN2m
slug: helm-chart
title: vfobs Helm chart — Deployment + Service + ESO ExternalSecret + ServiceMonitor
workgraph: foundation
order: 5
required_tags:
  - executor
depends_on:
  - t_W4kF6h
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t5-helm-chart 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T5-1 — `charts/vfobs/Chart.yaml` declares the chart name
    `vfobs`, `apiVersion: v2`, semver `version: 0.1.0` and
    `appVersion: "0.0.1"`. Maintainers + description present.
  - AC-T5-2 — `charts/vfobs/values.yaml` exposes:
    `image.repository`, `image.tag` (default `""` — falls back to
    `Chart.AppVersion`), `image.pullPolicy: IfNotPresent`,
    `replicaCount: 1`, `resources` (requests + limits), `env`
    (free-form list for non-secret env vars),
    `database.dsnSecret.name` + `database.dsnSecret.key` (k8s
    Secret reference for `VFOBS_DATABASE_URL`),
    `ingestToken.secret.name` + `ingestToken.secret.key` (k8s
    Secret reference for `VFOBS_INGEST_TOKEN`),
    `eso.enabled: true`, `eso.secretStore.name`,
    `eso.refreshInterval`, `monitoring.enabled: false`.
  - AC-T5-3 — `charts/vfobs/templates/deployment.yaml` renders a
    Deployment with: name `{{ include "vfobs.fullname" . }}`,
    replicas from values, container image
    `{{ .Values.image.repository }}:{{ .Values.image.tag | default
    .Chart.AppVersion }}`, env from `VFOBS_DATABASE_URL` (via
    secretKeyRef) + `VFOBS_INGEST_TOKEN` (via secretKeyRef), liveness
    probe `GET /healthz`, readiness probe `GET /readyz`. Probe
    timings sensible (initialDelaySeconds 5, periodSeconds 10).
    Resources from values. Container port 8080.
  - AC-T5-4 — `charts/vfobs/templates/service.yaml` renders a
    ClusterIP service exposing port 8080. Selector matches the
    Deployment.
  - AC-T5-5 — `charts/vfobs/templates/externalsecret.yaml` renders
    an `ExternalSecret` (external-secrets.io/v1beta1) iff
    `eso.enabled` is true. The ExternalSecret pulls
    `database-dsn` and `ingest-token` from the named
    SecretStore and writes them into the k8s Secret referenced by
    the Deployment.
  - AC-T5-6 — `charts/vfobs/templates/servicemonitor.yaml` renders
    a Prometheus-Operator ServiceMonitor iff
    `monitoring.enabled` is true (default off in dev; on in
    prod-flavored values).
  - AC-T5-7 — Integration:
    `tests/integration/test_helm_render.py` runs
    `helm template ./charts/vfobs --values tests/fixtures/values-dev.yaml`
    and asserts the rendered YAML contains a Deployment named
    `vfobs`, a Service, an ExternalSecret, and NO ServiceMonitor
    (default off). Second call with
    `tests/fixtures/values-monitoring.yaml` (sets
    `monitoring.enabled: true`) asserts the ServiceMonitor renders.
    Test parses rendered YAML with `yaml.safe_load_all`.
  - AC-T5-8 — Integration: same test file runs `helm lint
    ./charts/vfobs` and asserts exit code 0. Catches schema
    mistakes the renderer wouldn't.
  - AC-T5-9 — `charts/vfobs/templates/NOTES.txt` prints how to
    port-forward the service post-install and how to verify
    `/healthz` is reachable.
  - AC-T5-10 — Branch `wg-vfobs-foundation/t5-helm-chart` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `charts/vfobs/Chart.yaml` (new).
- `charts/vfobs/values.yaml` (new).
- `charts/vfobs/templates/_helpers.tpl` (new) — `vfobs.fullname`,
  `vfobs.labels`, `vfobs.selectorLabels`.
- `charts/vfobs/templates/deployment.yaml` (new).
- `charts/vfobs/templates/service.yaml` (new).
- `charts/vfobs/templates/serviceaccount.yaml` (new).
- `charts/vfobs/templates/configmap.yaml` (new) — non-secret env
  bundle (log level, service name).
- `charts/vfobs/templates/externalsecret.yaml` (new).
- `charts/vfobs/templates/servicemonitor.yaml` (new).
- `charts/vfobs/templates/NOTES.txt` (new).
- `charts/vfobs/.helmignore` (new) — standard.
- `tests/integration/test_helm_render.py` (new).
- `tests/fixtures/values-dev.yaml` (new).
- `tests/fixtures/values-monitoring.yaml` (new).

(NOT touched: app code under `src/vfobs/`. T5 packages the running
app from T4.)

## Implementation sketch

**Patterns: standard Helm chart conventions; no application-layer
patterns. Adapter pattern in the loose sense — the chart adapts the
running app to k8s primitives.**

### Chart.yaml

```yaml
apiVersion: v2
name: vfobs
description: ViloForge pipeline observability service
type: application
version: 0.1.0
appVersion: "0.0.1"
maintainers:
  - name: viloforge-platform
    email: lonvilo@pm.me
```

### values.yaml (annotated)

```yaml
image:
  repository: viloforge/vfobs
  tag: ""            # falls back to .Chart.AppVersion if empty
  pullPolicy: IfNotPresent

replicaCount: 1

resources:
  requests:
    cpu: 50m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

env: []              # free-form non-secret env

database:
  dsnSecret:
    name: vfobs-secrets
    key: database-url   # ESO writes this key

ingestToken:
  secret:
    name: vfobs-secrets
    key: ingest-token   # ESO writes this key

eso:
  enabled: true
  secretStore:
    name: vault-backend  # cluster's ESO SecretStore name
  refreshInterval: 1h

monitoring:
  enabled: false       # ServiceMonitor only if prometheus-operator is present
```

### deployment.yaml (representative shape)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "vfobs.fullname" . }}
  labels: {{- include "vfobs.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "vfobs.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "vfobs.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "vfobs.fullname" . }}
      containers:
        - name: vfobs
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: VFOBS_DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.database.dsnSecret.name }}
                  key:  {{ .Values.database.dsnSecret.key }}
            - name: VFOBS_INGEST_TOKEN
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.ingestToken.secret.name }}
                  key:  {{ .Values.ingestToken.secret.key }}
            {{- with .Values.env }}{{- toYaml . | nindent 12 }}{{- end }}
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            initialDelaySeconds: 5
            periodSeconds: 10
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

### externalsecret.yaml (gated by eso.enabled)

```yaml
{{- if .Values.eso.enabled }}
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: {{ include "vfobs.fullname" . }}
  labels: {{- include "vfobs.labels" . | nindent 4 }}
spec:
  refreshInterval: {{ .Values.eso.refreshInterval }}
  secretStoreRef:
    name: {{ .Values.eso.secretStore.name }}
    kind: SecretStore
  target:
    name: {{ .Values.database.dsnSecret.name }}
  data:
    - secretKey: {{ .Values.database.dsnSecret.key }}
      remoteRef:
        key: vfobs/database-url
    - secretKey: {{ .Values.ingestToken.secret.key }}
      remoteRef:
        key: vfobs/ingest-token
{{- end }}
```

### test_helm_render.py

```python
import subprocess, yaml, pytest
from pathlib import Path

CHART = Path(__file__).parent.parent.parent / "charts" / "vfobs"
DEV_VALUES = Path(__file__).parent / ".." / "fixtures" / "values-dev.yaml"
MON_VALUES = Path(__file__).parent / ".." / "fixtures" / "values-monitoring.yaml"

def _render(values: Path) -> list[dict]:
    out = subprocess.check_output(
        ["helm", "template", str(CHART), "--values", str(values)],
        text=True,
    )
    return list(yaml.safe_load_all(out))

@pytest.mark.integration
def test_dev_values_render():
    docs = [d for d in _render(DEV_VALUES) if d]
    kinds = {(d["kind"], d["metadata"]["name"]) for d in docs}
    assert ("Deployment", "vfobs") in {(k, n.replace("release-name-", "")) for k, n in kinds} \
        or any(k == "Deployment" for k, _ in kinds)
    assert any(d["kind"] == "Service" for d in docs)
    assert any(d["kind"] == "ExternalSecret" for d in docs)
    assert not any(d["kind"] == "ServiceMonitor" for d in docs)

@pytest.mark.integration
def test_monitoring_enabled_renders_servicemonitor():
    docs = [d for d in _render(MON_VALUES) if d]
    assert any(d["kind"] == "ServiceMonitor" for d in docs)

@pytest.mark.integration
def test_helm_lint():
    subprocess.check_call(["helm", "lint", str(CHART)])
```

## Fail-loud directive (R3)

If any required step (helm tooling missing, lint failure, branch
push) cannot be completed due to missing dependencies, missing
tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success. The judge will fail tasks that report
dishonest success; tasks that report failure honestly can be
reworked.

# Constraints

- Chart MUST NOT bundle Postgres — vfobs uses the shared vtaskforge
  Postgres per OIQ2. The chart only owns the vfobs Deployment +
  Service + the ESO secrets it consumes.
- The `vfobs-secrets` k8s Secret is **created by ESO**, not by
  Helm directly. Helm declares the ExternalSecret; ESO produces the
  Secret. Deployment references the resulting Secret.
- No Ingress in v1 — vfobs is cluster-internal. Operators reach it
  via `kubectl port-forward`. Future Ingress lands when WG2's
  read API + operator CLI go external.
- Probe paths MUST match T3: `/healthz` liveness, `/readyz`
  readiness. Drift here causes silent pod restarts.

# Out of scope

- ArgoCD ApplicationSet / Application manifest. That lives in
  `viloforge-platform/argo/products/vfobs/` and is a separate
  ops follow-up; chart-availability + chart-correctness are
  prerequisites that this task delivers.
- Vault secret seeding (the actual values for `vfobs/database-url`
  and `vfobs/ingest-token`). Operator-or-ops creates these in Vault
  before deploy.
- Image build CI. Out-of-scope per IMPLEMENTATION-PLAN §13.
  Operator builds and pushes the image manually for now.
- HorizontalPodAutoscaler. v1 is single-replica.
- NetworkPolicy. Cluster-wide policy is operator-managed; vfobs
  is not exempt.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §F, NFR4
- vtaskforge's helm chart in
  `vtaskforge/charts/vtf/` — reference layout
- External Secrets Operator docs:
  https://external-secrets.io/latest/api/externalsecret/
- Helm chart best practices:
  https://helm.sh/docs/chart_best_practices/
