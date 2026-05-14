# vfobs foundation (WG1)

Build the minimum-viable vfobs service that can accept POSTed
events from a controller and persist them in Postgres. Foundation
layer for v1 — WG2-5 build on top.

Canonical spec artifacts (do not duplicate the body in the
vtaskforge task records — link instead):

- workgraph.md — DAG, intent, workgraph-level ACs
- plan.md — D1-D8 design decisions, OIQ resolutions, alternatives
- tasks/00-postgres-schema.md through tasks/06-scenario-roundtrip.md
  — full task specs (file:line citations, fail-loud directives,
  pyramid-level ACs, etc.)
- verifier-findings.md — V1-V15 pass results (F1 blocking defect
  patched inline; six advisory findings recorded for retro)

Implementation discipline: per IMPLEMENTATION-PLAN §11, Claude (in
operator chat) implements manually under SDD discipline while
vafi-executor remains unreliable. When vafi-executor stabilizes,
the per-task vtaskforge records (this import) remain valid for
autonomous re-routing.
