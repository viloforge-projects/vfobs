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

## D2 — Controller hook points (verified against vafi source)

| Event | Where (factual) |
|---|---|
| `task.claimed` | `controller.py:_poll_and_execute`, right after `claimed_task = await self.work_source.claim(...)` (~L164) |
| `task.heartbeat` | `heartbeat.py:heartbeat_loop` per tick (the per-task loop, alongside the existing `work_source.heartbeat`) |
| `harness.turn_started` | `invoker.py:invoke`, before `_run_harness` |
| `harness.turn_completed` | `invoker.py:invoke`, after `_parse_harness_output` (carries turns-so-far if cheaply available) |
| `task.state_changed` (terminal, w/ `execution_summary`) | `controller.py:_poll_and_execute`, after `execute()` returns + result reported (~L189+) |

The judge path (`_poll_and_review`) gets the same claimed/terminal
pair in a later increment; this slice is the executor signals the
stuck-detection use case needs. No new turn counter is threaded
into heartbeat (see D7).

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
last_heartbeat_at, max harness-event id seen). Applies a Strategy
set of anomaly rules, prints a one-line live verdict each tick, and
exits non-zero / emits a clear ALERT line when a rule fires:

- `ApproachingTimeout` — `now - claimed_at > 0.8 * taskTimeout`
  and not terminal. (taskTimeout from config/flag; G2 default.)
- `Stall` — heartbeats still arriving (alive) BUT no new
  `harness.*` event for > `stall_seconds` (default 60s). This is
  the exact pi-hang cross-signal.
- `Crashed` — no heartbeat for > `crash_seconds` (default 120s).

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

## D7 — Progress signal without threading a turn counter

The watcher's "progressing?" check does NOT need a turn number
inside the heartbeat. It compares the **max id of `harness.*`
events** seen between polls: new harness events since last poll =
progressing; none while heartbeats continue = stalled. This keeps
the `heartbeat_loop` change to a single bare `task.heartbeat`
emit (minimal blast radius on a production loop) and puts the
cross-signal logic in the watcher where it's testable.

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
