# vfobs (ViloForge Pipeline Observability)

API-first observability service for the ViloForge AI agent fleet —
event ingestion, SSE streams, anomaly detection, cost rollups, and
workgraph DAG views.

This is the **project workspace** for vfobs work. It holds the
durable SDD record (decisions, gotchas, observations, workgraphs,
methodology overrides). The service code lives in
[`viloforge/vfobs`](https://github.com/viloforge/vfobs).

## Current focus

**WG0 scaffolding complete (2026-05-14). WG1 architect session next.**

The v1 implementation is decomposed into five sequential workgraphs
per `viloforge/viloforge-platform/docs/pipeline-observability-IMPLEMENTATION-PLAN.md`:

- **WG1** foundation — Postgres schema + write API + event types
- **WG2** read API + cost aggregation
- **WG3** SSE streams + workgraph DAG
- **WG4** anomaly detection (stuck-task detector)
- **WG5** SDK + CLI + vafi controller instrumentation + scenarios

WG0 (this scaffolding) was operator-bootstrapped without SDD ceremony
per the IMPLEMENTATION-PLAN §4 note.

Implementation pace per operator decision 2026-05-14: Claude (in
chat) implements manually while vafi-executor remains unreliable.
Per-task vtaskforge records still get created; the discipline
doesn't change, only who runs the code.

## Engineering discipline

Every workgraph applies the
[engineering north star](https://github.com/viloforge/viloforge-platform/blob/main/docs/engineering-principles.md)
v1.0 — SOLID + named design patterns + TDD red/green + full testing
pyramid up to scenario. Non-negotiable.

Per-role methodology files at
[`vtf-methodologies/<role>/bugfix.md`](https://github.com/viloforge/vtf-methodologies):

- `spec-author/bugfix.md` R1-R12
- `executor/bugfix.md` R1-R8
- `verifier/bugfix.md` V1-V15
- `judge/bugfix.md` R1-R8

## Reference docs (live in viloforge-platform)

- [`pipeline-observability-DESIGN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/pipeline-observability-DESIGN.md) — v0.2 locked architecture
- [`pipeline-observability-IMPLEMENTATION-PLAN.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/pipeline-observability-IMPLEMENTATION-PLAN.md) — v0.2 plan
- [`engineering-principles.md`](https://github.com/viloforge/viloforge-platform/blob/main/docs/engineering-principles.md) — v1.0 north star

## Empirical motivation

vfobs exists because the closed-system defect in the vafi pipeline
went undetected for 3 historical canary tasks. See
[`viloforge-projects/vafi/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md`](https://github.com/viloforge-projects/vafi/blob/main/spikes/canary-end-to-end-2026-05-14/SPIKE-RETRO.md)
for the empirical evidence and the three-layer defense pattern that
fed into the design.

## Linked code repos

- [`viloforge/vfobs`](https://github.com/viloforge/vfobs) — source
- [`viloforge/viloforge-platform`](https://github.com/viloforge/viloforge-platform) — deploy

## Layout

Follows `viloforge/viloforge-platform/docs/project-repo-DESIGN.md`:

```
.
├── README.md                       # this file
├── project.yaml                    # rigid project metadata
├── decisions/                      # durable architectural records
├── gotchas/                        # documented surprises
├── observations/                   # incoming raw signals
├── workgraphs/                     # work units (DAGs of tasks)
│   └── <slug>/
│       ├── workgraph.md
│       ├── plan.md
│       ├── tasks/
│       ├── reviews.md
│       └── completion.md
├── spikes/                         # exploratory probes outside the SDD flow
├── methodologies/                  # project-scoped methodology overrides
└── docs/                           # free-form project documentation
```

## Pointers

- Cycle: `2026-Q2-bootstrap`
- Methodology pin: `vtf-methodologies@v0.0`
- See `project.yaml` for structured metadata.
