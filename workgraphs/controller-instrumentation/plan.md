---
authored_by: claude-opus-4-7
co_authors:
  - operator:lonvilo
created_at: 2026-05-16T00:00:00Z
last_revision: 2026-05-16T00:00:00Z
---

# Approach

vfobs is queryable (WG1 write + WG2 read) but nothing emits. This
workgraph adds the producer (SDK + the minimum vafi controller
hooks) and a consumer (a watcher) that together deliver one
concrete capability: **see a stuck executor run early instead of
waiting for its timeout.**

# Design decisions

## D1 — SDK location & shape (NFR5)

`vfobs-sdk-python` lives **in the vfobs repo** at `sdk/python/`
(package `vfobs_sdk`), built like the existing `vafi/vtf-sdk-python`
precedent (setuptools, pydantic>=2 + httpx, `__version__`, semver).
Rationale: NFR5 — "the API ships with its SDK; the HTTP shape is an
SDK implementation detail." vafi consumes it as a dependency
(editable/path install for the bootstrap period; pip-published
later). **Rejected:** SDK vendored in vafi (couples the contract to
one consumer, violates NFR5); raw httpx calls in the controller
(no versioned boundary).

## D2 — Controller hook points (REVISED post T1 pre-impl review)

The first cut assumed harness turns are a mid-run progress signal
and that the controller carries `workgraph_id`. The pre-impl
review (`t1-preimpl-review.md`, verified against actual vafi
source — applies [[feedback-verified-fact-discipline]]) proved
both false. Revised, operator-ratified:

| Event | Where (verified source) | Notes |
|---|---|---|
| `task.claimed` | `controller.py:_poll_and_execute`, after `self.work_source.claim(...)` | `agent_id` = `self._agent_info.id` |
| `task.heartbeat` | `heartbeat.py:heartbeat_loop` per tick | the **alive** signal |
| `task.workdir_changed` | same `heartbeat_loop` tick, emitted **only when** a cheap workdir signature changed since the last tick | the **progress** signal (G2 fix) |
| `task.state_changed` (terminal, `execution_summary`) | `controller.py:execute()` after `invoker.invoke` returns | `success→to_status` mapped (D8) |
| `harness.turn_started` / `harness.turn_completed` | `controller.py:execute()`, bracketing the `invoker.invoke` call (NOT inside the opaque invoker) | **coarse phase markers only** — one pair per invocation; explicitly NOT the stall signal (G2) |

- **G1 — `workgraph_id`:** added to `TaskInfo` at the
  worksource→TaskInfo mapping boundary (vtf already associates a
  task to a workgraph; the mapping just stops dropping it). Every
  hook reads `task.workgraph_id`. Without this no event is valid
  (envelope requires it). See D8.
- **G2 — progress signal:** the harness runs as a buffered opaque
  subprocess (`communicate()` until exit) — there is *no* mid-run
  harness event. The DESIGN's own stuck criterion is *"workdir
  has not changed in N minutes."* So progress = `task.workdir_changed`
  recency, computed controller-side. See revised D7.
- The judge path (`_poll_and_review`) gets claimed/terminal in a
  later increment. `invoker.py` stays **pure** (no emitter, no
  ids) — G4.

## D3 — Fail-safe semantics (load-bearing, AC-gated)

The Emitter MUST be invisible to the controller's success path and
failure path alike:

- `emit(event)` **never raises** — all exceptions (network, 4xx,
  5xx, timeout, serialization) are caught.
- **Bounded**: per-emit hard timeout (default 2s, must be <
  heartbeat interval) so a hung vfobs cannot stall the loop.
- **Non-blocking delivery**: events go onto a bounded in-process
  queue (maxlen, drop-oldest on overflow) drained by a single
  background task; the controller call site does an O(1) enqueue,
  never awaits the HTTP.
- **Quiet failure**: first failure logs once at WARNING; repeats
  are rate-limited (no log spam if vfobs is down for an hour).
- **NullEmitter** when emission is unconfigured/disabled — a no-op
  implementation, selected at construction. This is the symmetric
  twin of D-T0-1 (read-side degradable): *emission is degradable
  too; observability is never on the critical path.*

## D4 — Watcher (`vfobs-watch`)

CLI: `vfobs-watch --task <id> [--workgraph <id>] [--interval 5]`.
Polls the WG2 read API; keeps per-task state (claimed_at,
last_heartbeat_at, **last `task.workdir_changed` at**). Applies a
Strategy set of anomaly rules, prints a one-line live verdict each
tick, and exits non-zero / emits a clear ALERT line when a rule
fires:

- `ApproachingTimeout` — `now - claimed_at > 0.8 * taskTimeout`
  and not terminal. (taskTimeout from config/flag; G2 default.)
- `Stall` — heartbeats still arriving (alive) BUT no
  `task.workdir_changed` for > `stall_seconds` (default 60s).
  **(REVISED per G2 — keys on workdir-change recency, not harness
  recency; the harness emits no mid-run events.)** This is the
  exact pi-hang cross-signal.
- `Crashed` — no heartbeat for > `crash_seconds` (default 120s).

> The merged T2 (PR #17) implemented `Stall` against harness
> recency. Task `04-watcher-stall-correction` re-points it at
> `task.workdir_changed` per this revision. The rule *shape*
> (alive-but-not-progressing) is unchanged — only its input event.

Each rule is one class implementing `evaluate(state) -> Verdict |
None`. WG4 later instantiates *these same rule classes*
server-side — the watcher is the rule library's first consumer,
not a throwaway.

## D5 — Emitted-envelope contract (V17 discipline)

The SDK serializes events using the WG1 event models (imported, or
a vendored copy). A contract test pins the SDK's emitted JSON for
each event type against WG1's locked
`tests/fixtures/event_schemas.v1.json`. Adding an emitter must not
drift the ingest schema (the V17 lesson from WG2-F1→R2, applied
preventively here).

## D6 — Enablement / config

Controller reads `VFOBS_EMIT_URL`, `VFOBS_EMIT_TOKEN`,
`VFOBS_EMIT_ENABLED` (default off until rolled out). Unset/false →
`NullEmitter`; the controller behaves exactly as today. Turning it
on is a config flip, reversible, no code change (IaC-friendly).

## D7 — Progress signal = workdir-change recency (REVISED, G2)

**Original (wrong):** "new `harness.*` events between polls =
progressing." The pre-impl review proved the harness is an opaque
buffered subprocess — there are no mid-run harness events, so that
rule false-positives `STALLED` on every healthy long task.

**Revised:** progress = **workdir change**, the DESIGN's own stuck
criterion. Each `heartbeat_loop` tick the controller computes a
*cheap* workdir signature for the task's workdir — `git -C
<workdir> status --porcelain` hashed, plus the max file mtime as a
fallback when not a git tree. If it changed since the previous
tick, emit `task.workdir_changed` (an existing WG1 event type:
`{files_changed, commits, branch}`-ish payload — populate what's
cheap, the *recency* is what matters). The watcher's `Stall` rule
keys on **`task.workdir_changed` recency**: heartbeats fresh
(alive) AND no workdir change for > `stall_seconds` = STALLED.
This is the exact pi-hang cross-signal, with a signal that is
actually observable mid-run without touching the opaque harness
subprocess.

Cost on the production loop: one `git status` per heartbeat
interval (default 30s) over a local workdir — negligible, and
wrapped by the fail-safe (D3): a slow/failing `git status` is
caught + the tick still emits the bare heartbeat.

**Downstream:** this changes an assumption baked into the
already-merged T2 watcher (PR #17 keyed `Stall` on harness
recency). A dedicated correction task (workgraph T-fix:
`tasks/04-watcher-stall-correction.md`) re-points T2's
`WatchState`/`Stall` at `task.workdir_changed`. T0 (emitter) is
signal-agnostic — unaffected.

## D8 — `workgraph_id` source + terminal-status mapping (G1, G5)

- `TaskInfo` gains `workgraph_id: str = ""` — a **default**, not
  a required field (verifier V16: vafi tests construct
  `TaskInfo(...)` directly; a required no-default field regresses
  them — the WG2 D-T0-1 class). Production worksources populate it
  from the vtf task↔workgraph link (`worksources/vtf.py`, which
  currently drops it). Empty ⇒ hooks skip-emit + log-once
  (degrade, never crash). `types.py` + worksource mapping change;
  in T1's blast radius by necessity (no event is valid without it).
- `task.state_changed` terminal mapping: `ExecutionResult.success
  is True → to_status="done"`; `False → to_status="failed"`
  (`from_status="doing"`). `ExecutionResult` has no `total_tokens`
  → `execution_summary.total_tokens=None` (Optional; do NOT widen
  vafi `types.py` for it in this slice — flagged for a later
  increment).

## D9 — Emitter lifecycle

The Emitter is constructed once via `make_emitter(config)` and
injected into `Controller`. `Controller.run()`'s `finally` block
MUST `await emitter.aclose()` so the bounded queue flushes the
last task's terminal `task.state_changed` on shutdown (otherwise
the most operationally-interesting event can be the one dropped).

# Considered alternatives

- **Alt A — instrument via raw httpx in the controller.** Rejected
  (NFR5; no versioned contract; every consumer re-implements).
- **Alt B — synchronous emit at each hook.** Rejected — adds vfobs
  latency/failure onto the production critical path (violates the
  D-T0-1 degradable principle). Hence D3's queue.
- **Alt C — thread a live turn counter into heartbeat.** Rejected
  for the minimal slice — bigger change to a hot production loop;
  D7 gets the same signal from harness event ids.
- **Alt D — build the server-side anomaly worker now.** That is
  WG4; doing it here couples a new write-path detector to this
  slice. The watcher proves the rules client-side first.

# Open methodology questions (for retro)

- **MQ1.** Background-queue overflow policy — drop-oldest vs
  drop-newest vs block-with-timeout. Default drop-oldest (lose the
  least-recent heartbeat, keep newest progress). Validate in retro.
- **MQ2.** SDK distribution: path/editable install during
  bootstrap; when does it become a pip-published artifact, and
  who owns the release? (NFR5 says versioned releases.)
- **MQ3.** Watcher thresholds are CLI/config defaults from DESIGN
  G2; empirical tuning belongs to WG4's retro once real runs flow.

# References

- workgraph.md (this dir) — DAG, WG-ACs, scope.
- DESIGN §3/§4(G1,G2)/NFR5.
- vafi `src/controller/{controller,heartbeat,invoker}.py`;
  `vtf-sdk-python/` packaging precedent.
- kb `jakdRrCh` (degradable principle — D3's basis); WG2
  `verifier-findings.md` V17 (D5's basis).
