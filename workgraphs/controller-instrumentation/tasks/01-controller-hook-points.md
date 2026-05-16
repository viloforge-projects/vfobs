---
id: t_ci_hook1
slug: controller-hook-points
title: vafi controller emission hook points (fail-safe)
workgraph: controller-instrumentation
order: 1
required_tags:
  - executor
depends_on:
  - t_ci_sdk0
target_repo: vilosource/vafi
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-ctrlinstr/t1-controller-hook-points 2>&1 | grep -q .
created_at: 2026-05-16T00:00:00Z
acceptance_criteria:
  - "AC-T1-1 — vafi depends on vfobs_sdk (path/editable for
    bootstrap); an Emitter is constructed once via
    make_emitter(config) and injected into Controller (DIP) —
    NullEmitter when VFOBS_EMIT_ENABLED is false/unset."
  - "AC-T1-2 — task.claimed emitted in controller.py
    _poll_and_execute immediately after work_source.claim()."
  - "AC-T1-3 — task.heartbeat emitted each tick of
    heartbeat.py:heartbeat_loop (alongside the existing
    work_source.heartbeat; bare heartbeat per plan §D7)."
  - "AC-T1-4 — harness.turn_started before _run_harness and
    harness.turn_completed after _parse_harness_output in
    invoker.py."
  - "AC-T1-5 — task.state_changed (terminal, data.execution_summary
    populated from the parsed result) after execute() completes in
    _poll_and_execute."
  - "AC-T1-6 — FAIL-SAFE (load-bearing): with the emitter pointed
    at an unreachable / 5xx / slow vfobs, a task still runs to its
    normal terminal result and the loop is not measurably slowed
    (no raise, no await-on-HTTP at any hook site)."
  - "AC-T1-7 — Unit: hooks call emitter.emit with the right event
    type+ids at each site (BufferingEmitter); a raising emitter
    double does NOT propagate out of the controller/invoker."
  - "AC-T1-8 — Integration: a full claim→execute→terminal cycle
    against a stub work source + BufferingEmitter yields the
    expected ordered event sequence; same cycle with an
    always-raising emitter completes identically (fail-safe)."
  - "AC-T1-9 — Existing vafi controller test suite green (zero
    regression); ruff/type-clean per vafi's gates."
  - "AC-T1-10 — Branch wg-vfobs-ctrlinstr/t1-controller-hook-points
    pushed to vilosource/vafi + PR open."

---

# Spec

## Files touched
- `vafi/pyproject.toml` (add vfobs_sdk dep, path/editable).
- `vafi/src/controller/controller.py` (claimed + terminal hooks;
  construct/inject emitter).
- `vafi/src/controller/heartbeat.py` (heartbeat hook).
- `vafi/src/controller/invoker.py` (turn hooks).
- `vafi/src/controller/config.py` (VFOBS_EMIT_* config).
- `vafi/tests/...` (new unit + integration per AC-T1-7/8).
- (NOT touched: worksources, gates, judge review path — later
  increment; existing tests unmodified except additive.)

## Implementation sketch

**Pattern: DIP — Controller/Invoker hold an `Emitter` (the T0
ABC), never httpx. Emission is a side-effect call, never awaited
on the critical path (T0's emit() is O(1) by construction).**

```python
# controller.py
claimed = await self.work_source.claim(task.id, agent.id)
self._emitter.emit(EventFactory.task_claimed(
    workgraph_id=claimed.workgraph_id, task_id=claimed.id,
    source=f"vafi-controller/{agent.id}", claimed_by_agent_id=agent.id))
...
result = await self.execute(claimed)
self._emitter.emit(EventFactory.task_state_changed(
    workgraph_id=claimed.workgraph_id, task_id=claimed.id,
    source=..., from_status="doing", to_status=result.terminal_status,
    execution_summary=ExecutionSummary(**result.summary)))
```

Hook sites are exactly those verified in plan.md §D2. Emission is
wrapped nowhere special — T0 guarantees emit() never raises; the
AC-T1-7/8 raising-double test is the belt-and-braces proof.

## Fail-loud directive (R3)

If the vfobs_sdk dependency wiring, a hook site, or the fail-safe
proof cannot be completed: report explicitly — do not mark done
with the fail-safe test skipped or weakened.

# Constraints
- This modifies a PRODUCTION control loop. Default OFF
  (NullEmitter) — turning emission on is a config flip.
- No hook may add an `await` on network I/O or a try/except that
  changes control flow on emit failure.
- Zero behavior change when emission is disabled (prove via the
  existing suite staying green with default config).

# Out of scope
- Judge/gate/workdir events; turn-counter threading (plan §D7).
- The watcher (T2). Anomaly worker (WG4).

# References
- plan.md §D2, §D3, §D6, §D7. T0 emitter contract.
- vafi `src/controller/{controller,heartbeat,invoker,config}.py`.
- `vtf-methodologies/executor/bugfix.md` (incl. SC-1/SC-2).
