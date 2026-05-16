---
authored_by: claude-opus-4-7
created_at: 2026-05-16T00:00:00Z
workgraph: read-api
methodology_targets:
  - vtf-methodologies/verifier/bugfix.md
  - vtf-methodologies/executor/bugfix.md
---

# WG2 read-api — executor retro

Rolling-synthesis learnings for `vtf-methodologies` propagation
(per [[feedback-lab-notebook-propagation]] — patch SOPs as
findings emerge, not at workgraph end; captured here continuously).

## Verifier-methodology gaps (V1–V15 missed two classes)

**VR-1 — required-config AC vs frozen direct-construct fixtures.**
AC-T0-6 mandated a strictly-required `Settings.vtaskforge_url`
("app fails to start" if unset). V1–V15 has no check that catches
"a newly-required config field will break existing tests that
construct the config object directly." WG1 had ~67 such fixtures
(`Settings(database_url=..., ingest_token=...)`) + lifespan booted
via `with TestClient`. Surfaced at *implementation* (D-T0-1), not
verify. **Propose new verifier check:** for any task adding a
required field to a settings/config model, grep the target repo
for direct constructions of that model in tests; if the new field
has no default, flag it (reject or require a default + a
production-resolution guard).

**VR-2 — inline field-add to a model backing a locked contract.**
F1's inline patch added a declared `id` to the base `Event`. That
model's serialized JSON schema is a *locked contract fixture*
(`event_schemas.v1.json`). The verifier patched the model without
checking whether its `model_json_schema()`/`model_dump()` output
is pinned anywhere. Cost: a full mechanism reversal (R2) mid-T1.
**Propose new verifier check:** before an inline patch that mutates
a Pydantic model's fields, grep for `*.v1.json` / schema-pin
contract tests over that model; if found, the patch must either
bump the contract (in the same change) or use a separate read
model. (This is the F4/F1 pattern-discipline lineage extended.)

Both also argue the general principle: **inline verifier patches
that touch a shared/locked artifact must be validated against that
artifact in the same pass**, not deferred to the executor.

## Executor deviations (documented, all ratified)

- **D-T0-1** — read-side is degradable, not boot-critical;
  AC-T0-6's boot-crash guarantee → readiness concern. Operator
  ratified A. The *long-term-correct* framing (recorded in kb
  `jakdRrCh`): an ingest sink must not couple write-path boot to
  read-side config; loud-not-fatal + health/readiness separation.
- **D-T1-1** — `get_by_id` stays a bare `Event` (no id); only
  `find_*` wrap `StoredEvent`. Preserves the WG1 LSP contract +
  whole-object-equality tests; satisfies F1's intent.
- **D-T2-1** — LSP-tested helpers ⇒ contract ⇒ ABC membership;
  the spec's "private, ABC stays four" was scoped to not bloating
  T1, not a prohibition on T2 contract methods.

## Operational findings

- **OF-1 (prepare.sh kubeconfig isolation).** `scripts/scenario/
  prepare.sh` assumed the ambient kubeconfig holds the
  `kind-vfobs-scenario` context. On this workstation
  `~/.kube/config` is a **symlink to the viloforge *production*
  cluster** kubeconfig, and WG1's cleanup had deleted the kind
  context. A naive `kind export kubeconfig` would have written
  into the prod cluster's config (kb gotcha `bkt4wMJZ` + a
  prod-config footgun). Mitigation: ran the scenario with an
  isolated `KUBECONFIG=/tmp/vfobs-scenario.kubeconfig`. **Propose:
  prepare.sh should `kind export kubeconfig` to a dedicated file
  and export `KUBECONFIG` for itself + the pytest run** — never
  trust the ambient kubeconfig, especially when it may be a prod
  symlink. Executor-methodology hardening for kind scenarios.
- **OF-2 (shared scenario DB isolation).** The scenario cluster
  Postgres persists across test runs; task-scoped reads
  (`/tasks/<id>`, `find_by_task`) are global. Scenario tests must
  uuid-namespace **every** id they assert counts on — workgraph
  *and* task (T5 initially namespaced only workgraph; caught on
  the 2nd run). Same class as the integration `uuid` fix; promote
  to executor SOP for any test against a persistent shared store.

## What went well (keep)

- Verifier-first caught F1 + F2 *before* implementation; F2's
  de-dup avoided a real cost-double-count under rework.
- Per-task branch + PR + full pyramid + standing merge-auth kept
  the DAG flowing without per-PR approval stalls while still
  surfacing every *design* decision for ratification.
- DTO schema-pin contract tests (`reads_response_shapes.v1.json`)
  extended per task — cheap drift insurance for the read API.
