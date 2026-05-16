---
authored_by: claude-opus-4-7
created_at: 2026-05-16T00:00:00Z
scope: T1 (controller-hook-points) pre-implementation review
verdict: NOT ready — 2 blocking (G1, G2) require an architect decision
---

# T1 pre-implementation review

Requested by the operator before authorizing the production
`vilosource/vafi` controller change. Reviewed against the ACTUAL
vafi source (`src/controller/{controller,heartbeat,invoker,
types}.py`), not the grep-level confirmation the same-session
architect/verifier passes used. Two blocking gaps; three smaller
gaps; one positive confirmation.

## G1 — BLOCKING: the controller has no `workgraph_id`

`types.py:TaskInfo` = `{id, title, spec, project_id, needs_review,
assigned_to, session_id, judge_feedback, attempt_number}`. There
is **no `workgraph_id`** (nor on `ExecutionResult`, nor anywhere
the controller threads). Every vfobs event's envelope **requires**
`workgraph_id` (`Event.workgraph_id: str = Field(min_length=1)`).

Plan §D2's sketch `EventFactory.task_claimed(workgraph_id=
claimed.workgraph_id, …)` references an attribute that **does not
exist**. T1 cannot emit a single valid event as specced. The
premise that T1 is "add ~4 emit() calls" is wrong — the controller
does not currently carry the dimension vfobs is keyed on.

**Resolution (architect decision needed):**
- (a) derive `workgraph_id` from the task `spec` YAML, or
- (b) the work_source/vtf API exposes a task→workgraph link the
  controller must fetch + thread, or
- (c) add `workgraph_id` to `TaskInfo` at the worksource mapping
  boundary (where vtf task JSON → TaskInfo).
(c) is cleanest but widens T1's blast radius into `types.py` +
every worksource. This must be decided + a spec amendment written
before implementation.

## G2 — BLOCKING: no mid-run progress signal (opaque harness)

`invoker._run_harness` runs the harness via
`asyncio.create_subprocess_exec` + `await wait_for(
process.communicate(), timeout=task_timeout)`. `communicate()`
**buffers all stdout until the process exits.** Per-turn data
(`num_turns`, Pi's `turn_end`) only exists *after* exit, in
`_parse_harness_output`.

So plan §D2's hooks emit `harness.turn_started` **once** at t=0
and `harness.turn_completed` **once** at terminal. **There is no
harness event between the start and end of a (possibly 30-minute)
run.**

The **already-merged T2 watcher** (`Stall` rule) keys "progress"
on `last_harness_at` recency (plan §D7). With this architecture it
would flag **STALLED on every healthy long task within
`stall_seconds`** — a false positive that defeats the entire
"proactive instead of timeout" purpose. The DESIGN itself
anticipated this: its stuck criteria were *"workdir has not
changed in N minutes"* and *"cxdb has no live session"* — NOT
harness turns, precisely because the harness subprocess is opaque.
The architect (this session) over-indexed on harness turns.

**Resolution (architect decision needed) — the progress signal
must change to something observable mid-run WITHOUT restructuring
the opaque subprocess:**
- **Recommended:** a controller-side **workdir-change** signal —
  the heartbeat path computes a cheap workdir signature (e.g.
  `git status --porcelain` hash / mtime-max) and emits
  `task.workdir_changed` (a WG1 event type that already exists)
  when it changes. Matches the DESIGN's own criterion, is
  controller-side (never touches the opaque subprocess), bounded,
  fail-safe-compatible. The watcher's `Stall` rule then keys on
  workdir-change recency, not harness-event recency.
- Alternatives: stream harness stdout (harness-specific, only
  helps Pi JSONL not Claude `--output-format json`; risky change
  to the production invoker) or poll cxdb (out of WG5-min scope).

**Downstream impact:** G2 invalidates an assumption already in
**merged T2** (PR #17). Fixing it requires (i) a plan §D7 +
workgraph revision, (ii) a T1 spec that adds the workdir signal,
(iii) a follow-up PR amending the merged T2 `Stall` rule +
`WatchState` to key on workdir-change. T0 (emitter) is
signal-agnostic — unaffected.

## Smaller gaps

- **G3.** `heartbeat_loop(work_source, task_id, interval_seconds)`
  has no `workgraph_id` and no emitter handle. Signature + its
  single call site must change (subsumed by G1 for workgraph_id;
  still needs the emitter threaded in + an injected source string).
- **G4.** `invoker.invoke(task, repo, workdir, prompt)` has only
  `TaskInfo` — no `workgraph_id`, no `agent_id`. Emit the
  invoke-boundary events from `controller.execute()` (which has
  `self._agent_info.id`) rather than inside the invoker, keeping
  the invoker pure. (Consistent with G2 — there are no per-turn
  events to emit from inside it anyway.)
- **G5.** `ExecutionResult` = `{success, session_id,
  completion_report, cost_usd, num_turns, gate_results}` — **no
  `total_tokens`** (Pi computes it then discards it) and
  `success: bool` only (no terminal status string). AC-T1-5
  ("execution_summary populated from the parsed result")
  underspecifies: define the `success → to_status` mapping
  (doing→done | doing→changes_requested/failed) and accept
  `total_tokens=None` (or add it to `ExecutionResult` — scope
  creep into vafi types; flag, don't silently do it).

## Positive confirmation

- **G6 OK.** All hook sites (`_poll_and_execute`, `heartbeat_loop`,
  `execute`) are async with a running loop, so `HttpEmitter`'s
  lazy background-drain works. **But** `controller.run()`'s
  `finally` must `await emitter.aclose()` so the last task's
  terminal `task.state_changed` flushes on shutdown — add to the
  T1 spec (not currently called out).

## Recommendation

T1 is **NOT ready to implement**. G1 + G2 are design-level
(per verifier methodology, structural defects → architect, not
inline executor patches), and G2 reopens merged T2. Required
before T1:

1. Architect decision on G1 (workgraph_id source) + G2 (workdir
   progress signal), with a `plan.md`/`workgraph.md` revision.
2. A T1 spec amendment (G3/G4/G5/G6 fold in once G1/G2 resolved).
3. A planned follow-up to amend the merged T2 `Stall`/`WatchState`
   for the workdir signal — so the watcher is actually correct.

This review is the gate working as intended: the fast
same-session passes missed that "instrument the controller" is a
design problem (the controller lacks workgraph_id and the harness
is opaque), not a mechanical hook insertion.
