# vfobs read-api + cost aggregation (WG2)

Add the read side of vfobs: query endpoints the operator CLI, retro
tooling, and future web UI use to inspect the event log, plus cost
rollups computed from `execution_summary` payloads already arriving
via WG1-T4. Builds on the WG1 foundation (event store + write API).

Canonical spec artifacts (do not duplicate the body in the
vtaskforge task records — link instead):

- workgraph.md — DAG, intent, workgraph-level ACs (WG-AC1..AC6)
- plan.md — D1-D6 design decisions, OIQ3 read-half resolution,
  considered alternatives, open methodology questions
- tasks/00-vtf-adapter-and-read-auth.md through
  tasks/05-scenario-read-side.md — full task specs (file:line
  citations, fail-loud directives, pyramid-level ACs)
- verifier-findings.md — V1-V15 pass: TWO blocking defects
  (F1 Event.id propagation, F2 cost-aggregation de-dup) patched
  inline with empirical/authoritative grounding; F-adv1/F-adv4
  patched inline; F-adv2/F-adv3 carried to retro.
  Verdict: ready-after-inline-patches.

Implementation discipline: per IMPLEMENTATION-PLAN §11, Claude (in
operator chat) implements manually under SDD discipline while
vafi-executor remains unreliable. When vafi-executor stabilizes,
the per-task vtaskforge records (this import) remain valid for
autonomous re-routing. Task IDs mirror the spec-file frontmatter
ids 1:1 (t_v2adp1 … t_v2scen) for spec↔vtf traceability.
