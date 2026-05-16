---
id: t_ci_scen3
slug: scenario-proactive-loop
title: Scenario — proactive stuck-detection end-to-end
workgraph: controller-instrumentation
order: 3
required_tags:
  - executor
depends_on:
  - t_ci_hook1
  - t_ci_watch2
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-ctrlinstr/t3-scenario-proactive-loop 2>&1 | grep -q .
created_at: 2026-05-16T00:00:00Z
acceptance_criteria:
  - "AC-T3-1 — A scenario harness runs an instrumented controller
    (T1) against a STUB harness invoker whose behaviour is
    scripted: a 'healthy' task (emits turns) and a 'stuck' task
    (claims, heartbeats, but emits NO harness turns) — both
    emitting into the kind-deployed vfobs (WG2/T5 cluster + the
    vtfstub sidecar, isolated KUBECONFIG per executor SC-1)."
  - "AC-T3-2 — vfobs-watch (T2) run against the STUCK task prints
    `STALLED` and exits non-zero BEFORE the task's configured
    timeout would have elapsed (the proactive proof — WG-AC4)."
  - "AC-T3-3 — vfobs-watch against the HEALTHY task stays `OK` and
    exits zero at terminal."
  - "AC-T3-4 — A near-timeout task is flagged APPROACHING_TIMEOUT
    before its timeout."
  - "AC-T3-5 — Fail-safe end-to-end: with vfobs scaled to 0 /
    unreachable, the instrumented controller still drives both
    stub tasks to their normal terminal result (the loop is
    unaffected by emission being dead)."
  - "AC-T3-6 — docs/scenario-test-runbook.md gains a
    controller-instrumentation section (how to run the proactive
    loop; the isolated-KUBECONFIG reminder per SC-1)."
  - "AC-T3-7 — Branch pushed + PR open. FAIL-LOUD INTENSIFIER
    (WG1-T6/WG2-T5 precedent): NOT done unless this runs green
    end-to-end on the cluster."

---

# Spec

## Files touched
- `tests/scenario/test_proactive_loop.py` (the 3 outcomes +
  fail-safe).
- `scripts/scenario/stub_executor/` — a tiny scripted controller
  driver + stub harness invoker (healthy / stuck / near-timeout
  behaviours) reusing the T1 emitter wiring.
- `scripts/scenario/prepare.sh` — extend to (optionally) run the
  proactive-loop fixtures; **and** fix SC-1: self-isolate
  KUBECONFIG to a dedicated file (carried methodology item from
  WG2-T5 OF-1 — do it here).
- `docs/scenario-test-runbook.md` (update).

## Implementation sketch

**No new pattern — composes T1 (instrumented controller) + T2
(watcher) + the WG2/T5 kind harness. The stub harness invoker is
the seam that makes 'stuck' deterministic (it just sleeps without
emitting harness turns while heartbeats continue).**

The scenario asserts the WHOLE point of this workgraph: a stuck
run is caught by watching, not by waiting for the timeout.

## Fail-loud directive (R3)

Do NOT mark complete if the scenario does not run green
end-to-end on the cluster. In particular AC-T3-2 (STALLED before
timeout) and AC-T3-5 (fail-safe) are not waivable.

# Constraints
- Reuse the WG2/T5 cluster + vtfstub; isolated KUBECONFIG
  (executor SC-1) — and harden prepare.sh to do that itself.
- The 'stuck' stub MUST be deterministic (scripted), not timing-
  flaky; assertions key off events in vfobs, not wall-clock races.

# Out of scope
- Real Claude harness runs (stub invoker is sufficient + stable
  for the contract; real-harness canary is later).
- Auto-intervention; anomaly worker (WG4).

# References
- plan.md (all §). workgraph.md WG-AC3/AC4.
- WG2-T5 scenario harness; `vtf-methodologies/executor/bugfix.md`
  SC-1/SC-2 (kubeconfig isolation + uuid namespacing).
