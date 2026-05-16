---
id: t_ci_t2fix
slug: watcher-stall-correction
title: Correct the merged watcher Stall rule — workdir-change recency
workgraph: controller-instrumentation
order: 4
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-ctrlinstr/t4-watcher-stall-correction 2>&1 | grep -q .
created_at: 2026-05-16T00:00:00Z
acceptance_criteria:
  - "AC-T4-1 — WatchState.from_events tracks last_workdir_changed_at
    (max timestamp of task.workdir_changed events) instead of
    last_harness_at. harness.* events no longer drive the progress
    signal (the harness emits no mid-run events — pre-impl review
    G2)."
  - "AC-T4-2 — Stall.evaluate keys on workdir-change recency:
    heartbeat fresh (< crash_s) AND no task.workdir_changed for >
    stall_s ⇒ STALLED. Rule SHAPE (alive-but-not-progressing) and
    priority order (Crashed > Stall > ApproachingTimeout)
    unchanged — only the input event changes."
  - "AC-T4-3 — fallback: if NO task.workdir_changed yet, progress
    age is measured from claimed_at (a task that never touches its
    workdir within stall_s is correctly STALLED — that IS the
    pi-hang)."
  - "AC-T4-4 — Unit: tests/unit/test_watch_rules.py updated — the
    pi-hang case is now 'alive + heartbeats + NO workdir change';
    a healthy long task with periodic workdir changes but NO
    harness events is OK (the exact false-positive the old rule
    had); 0.79/0.81 + priority + terminal cases retained."
  - "AC-T4-5 — Integration: tests/integration/test_watch_live.py
    updated — seeded scenarios use task.workdir_changed for the
    progress dimension; healthy/stalled/near-timeout verdicts hold."
  - "AC-T4-6 — Full SDK suite green (no regression to T0/T2
    survivors); ruff clean."
  - "AC-T4-7 — Branch wg-vfobs-ctrlinstr/t4-watcher-stall-correction
    pushed + PR open."

---

# Spec

## Files touched
- `sdk/python/vfobs_sdk/watch.py` (WatchState fields +
  Stall.evaluate input; harness fields may stay as informational
  display but no longer gate Stall).
- `sdk/python/tests/unit/test_watch_rules.py` (rewrite the
  progress-dimension cases).
- `sdk/python/tests/integration/test_watch_live.py` (seed
  task.workdir_changed instead of harness.* for progress).

## Implementation sketch

**No new pattern — a corrective edit to the merged T2 Strategy.
The `Stall` class stays; its `evaluate` reads
`s.last_workdir_changed_at or s.claimed_at` where it previously
read `s.last_harness_at or s.claimed_at`. `WatchState.from_events`
swaps the `elif e.type.startswith("harness.")` progress branch for
`elif e.type == "task.workdir_changed"`.**

This is deliberately small + behavior-preserving for the rule
shape: the only conceptual change is *which event means
"progress"*, corrected from an architecturally-impossible signal
(harness mid-run) to the DESIGN's actual one (workdir change).

## Fail-loud directive (R3)

If the rule cannot be corrected without weakening a retained case
(0.79/0.81, priority, terminal short-circuit), or the SDK suite
regresses: report explicitly — do not mark done.

# Constraints
- Pure-function rules preserved (WG4 server-side reuse intact).
- This is a correction to a MERGED deliverable (PR #17); the
  commit message + PR must reference the pre-impl-review G2 as the
  cause, not present it as new work.

# Out of scope
- Any new rule. The emitter/controller side (T1). WG4.

# References
- t1-preimpl-review.md G2 (the defect this corrects).
- plan.md §D4/§D7 (revised). Merged T2: PR #17.
- DESIGN §3/§4 G2 ("workdir has not changed in N minutes").
