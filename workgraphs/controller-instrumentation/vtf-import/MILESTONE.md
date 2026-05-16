# vfobs controller-instrumentation (WG5-minimal)

Close the observability loop: make the vafi pipeline EMIT into
vfobs + give the operator a live watcher that flags a stuck
executor BEFORE its timeout (DESIGN §3 pi-hang failure). Minimal
slice: versioned vfobs-sdk Emitter + the 4 controller hook points
+ a client-side G2 watcher. Full anomaly worker = WG4.

Canonical spec artifacts (do not duplicate the body — link):
- workgraph.md — DAG, intent, WG-AC1..AC6, scope/out-of-scope
- plan.md — D1-D7 (SDK boundary, hook sites, fail-safe, watcher),
  considered alternatives, methodology questions
- tasks/00..03 — full task specs
- verifier-findings.md — V1-V17 pass (ready; F-adv1 patched
  inline; no blocking)

Implementation discipline (operator-agreed 2026-05-16): T0
(vfobs-sdk) + T2 (vfobs-watch) are greenfield viloforge/vfobs —
implemented autonomously + self-merged on green (standing auth).
**T1 (PRODUCTION vilosource/vafi controller) + T3 (depends on T1)
are GATED** — explicit go-ahead + a stated vafi merge policy
required before implementation. Task IDs mirror the spec-file
frontmatter ids 1:1 (t_ci_sdk0 … t_ci_scen3).
