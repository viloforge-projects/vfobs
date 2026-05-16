---
id: t_ci_hook1
slug: controller-hook-points
title: vafi controller emission hook points (fail-safe) — REVISED
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
    bootstrap); ONE Emitter built via make_emitter(config) is
    injected into Controller (DIP); NullEmitter when
    VFOBS_EMIT_ENABLED unset/false. Controller.run() finally
    `await emitter.aclose()` so the queue flushes on shutdown (D9)."
  - "AC-T1-2 — G1 (+ verifier V16): TaskInfo gains
    `workgraph_id: str = \"\"` — a DEFAULT, not a required field.
    Rationale: vafi constructs TaskInfo(...) directly in
    tests/test_controller.py, test_invoker.py, test_gates.py; a
    required no-default field regresses them at construction (the
    WG2 D-T0-1 / V16 class). Production worksources
    (src/controller/worksources/vtf.py) populate it from the vtf
    task↔workgraph link (the mapping currently drops it). Hooks
    SKIP emit + log-once when workgraph_id is empty (degrade,
    never crash — symmetric to the fail-safe philosophy). ACs:
    (a) vtf worksource maps workgraph_id (unit-asserted, no longer
    dropped); (b) empty workgraph_id ⇒ no emit, no raise
    (unit-asserted); (c) existing vafi suite green at default
    construction."
  - "AC-T1-3 — task.claimed emitted in controller.py
    _poll_and_execute after work_source.claim(); agent_id =
    self._agent_info.id."
  - "AC-T1-4 — task.heartbeat emitted each heartbeat_loop tick.
    heartbeat_loop signature gains workgraph_id + emitter (+ a
    workdir path); its single call site updated. (G3)"
  - "AC-T1-5 — G2 PROGRESS SIGNAL: each heartbeat_loop tick
    computes a cheap workdir signature (`git -C <wd> status
    --porcelain` hashed; max-mtime fallback). On change since the
    prior tick, emit task.workdir_changed. Unbounded/again: the
    signature computation is wrapped fail-safe (a slow/failed git
    still emits the bare heartbeat)."
  - "AC-T1-6 — task.state_changed (terminal) emitted in
    controller.py execute() after invoker.invoke returns;
    success→to_status: True→done, False→failed; from_status=doing;
    execution_summary={num_turns, cost_usd, total_tokens:None}
    from ExecutionResult (D8 — total_tokens absent there; Optional,
    do NOT widen vafi types)."
  - "AC-T1-7 — harness.turn_started/completed emitted in
    controller.py execute() BRACKETING the invoker.invoke call
    (NOT inside the pure invoker — G4). Documented as COARSE phase
    markers (one pair per invocation), explicitly NOT the stall
    signal."
  - "AC-T1-8 — FAIL-SAFE (load-bearing): with the emitter pointed
    at unreachable / 5xx / slow vfobs, a task still runs to its
    normal terminal result and the loop is not measurably slowed
    (no raise, no await-on-HTTP, no workdir-sig stall at any hook)."
  - "AC-T1-9 — Unit: hooks call emit with the right event type +
    workgraph_id/task_id/agent_id at each site (BufferingEmitter);
    a raising emitter double does NOT propagate out of
    controller/heartbeat; workdir-sig change detection unit-tested."
  - "AC-T1-10 — Integration: a full claim→execute→terminal cycle
    vs a stub work source + BufferingEmitter yields the expected
    ordered sequence incl. task.workdir_changed when the workdir
    mutates; same cycle with an always-raising emitter completes
    identically (fail-safe)."
  - "AC-T1-11 — Existing vafi controller suite green (zero
    regression); vafi ruff/type gates clean."
  - "AC-T1-12 — Branch wg-vfobs-ctrlinstr/t1-controller-hook-points
    pushed to vilosource/vafi + PR open."

---

# Spec

## Files touched
- `vafi/pyproject.toml` (add vfobs_sdk dep, path/editable).
- `vafi/src/controller/types.py` (G1 — `TaskInfo.workgraph_id`).
- `vafi/src/controller/worksources/*` (G1 — populate
  workgraph_id in the task mapping; every impl).
- `vafi/src/controller/controller.py` (claimed + terminal +
  harness-bracket hooks; build/inject emitter; aclose in finally).
- `vafi/src/controller/heartbeat.py` (G3 — signature +
  heartbeat + workdir-change emit).
- `vafi/src/controller/config.py` (VFOBS_EMIT_* config).
- `vafi/tests/...` (additive unit + integration per AC-9/10).
- (NOT touched: `invoker.py` stays pure — G4; gates/judge path —
  later increment.)

## Implementation sketch

**Patterns: DIP (Controller/heartbeat hold the T0 `Emitter` ABC,
never httpx). Emission is a side-effect call, never awaited
(T0 `emit()` is O(1) by construction).**

```python
# G1: types.py
@dataclass
class TaskInfo:
    id: str
    workgraph_id: str   # NEW — mapped from the vtf task, not dropped
    ...

# controller._poll_and_execute
claimed = await self.work_source.claim(task.id, self._agent_info.id)
self._emitter.emit(task_claimed(
    workgraph_id=claimed.workgraph_id, task_id=claimed.id,
    source=f"vafi-controller/{self._agent_info.id}",
    claimed_by_agent_id=self._agent_info.id))

# controller.execute (brackets the opaque invoker — G4/G2)
self._emitter.emit(harness_turn_started(workgraph_id=wg, task_id=tid,
    source=src, turn_number=0, model=self.config.harness))
result = await self._invoker.invoke(...)
self._emitter.emit(harness_turn_completed(workgraph_id=wg,
    task_id=tid, source=src, turn_number=result.num_turns))
self._emitter.emit(task_state_changed(
    workgraph_id=wg, task_id=tid, source=src,
    from_status="doing",
    to_status="done" if result.success else "failed",
    execution_summary=ExecutionSummary(
        num_turns=result.num_turns, cost_usd=result.cost_usd)))

# heartbeat_loop(work_source, task_id, *, workgraph_id, workdir,
#                emitter, interval) — G3/G2
prev_sig = None
while True:
    await asyncio.sleep(interval)
    await work_source.heartbeat(task_id)          # unchanged
    emitter.emit(task_heartbeat(workgraph_id=workgraph_id,
        task_id=task_id, source=src))             # alive
    sig = _workdir_sig(workdir)                   # fail-safe wrapped
    if sig is not None and sig != prev_sig:
        prev_sig = sig
        emitter.emit(task_workdir_changed(workgraph_id=workgraph_id,
            task_id=task_id, source=src, files_changed=..., ...))
```

`_workdir_sig`: `git -C <wd> status --porcelain` → sha256; on any
error/timeout return None (skip the workdir emit; never break the
heartbeat). The whole tick is inside the existing
try/except-that-doesn't-crash-the-loop.

## Fail-loud directive (R3)

If vfobs_sdk wiring, the G1 workgraph_id plumbing, a hook, or the
fail-safe proof cannot be completed: report explicitly — do NOT
mark done with the fail-safe or workgraph_id mapping test
skipped/weakened.

# Constraints
- Modifies a PRODUCTION control loop. Default OFF (NullEmitter);
  turning emission on is a config flip. Zero behavior change when
  disabled (prove via the existing suite green at default config).
- No hook may add an `await` on network I/O or change control flow
  on emit/workdir-sig failure.
- G1 is non-negotiable: no vfobs event is valid without
  workgraph_id; the worksource mapping change is in scope.

# Out of scope
- Judge/gate/workdir-diff *content* events; per-turn harness
  events (architecturally impossible — opaque subprocess; G2).
- The watcher correction (separate task 04). Anomaly worker (WG4).

# References
- plan.md §D2/§D3/§D7/§D8/§D9 (revised). t1-preimpl-review.md
  (G1-G6, verified vs source — [[feedback-verified-fact-discipline]]).
- vafi `src/controller/{controller,heartbeat,invoker,types,
  worksources}` (read, not grepped).
- `vtf-methodologies/executor/bugfix.md` (incl. SC-1/SC-2).
