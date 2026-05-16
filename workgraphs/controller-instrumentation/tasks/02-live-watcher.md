---
id: t_ci_watch2
slug: live-watcher
title: vfobs-watch — live proactive stuck-detection CLI
workgraph: controller-instrumentation
order: 2
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-ctrlinstr/t2-live-watcher 2>&1 | grep -q .
created_at: 2026-05-16T00:00:00Z
acceptance_criteria:
  - "AC-T2-1 — `vfobs-watch --task <id> [--workgraph <id>]
    [--interval 5] [--task-timeout S]` polls the WG2 read API
    (/tasks/<id>/events + /events?filter) and prints a one-line
    verdict per tick: state, last-heartbeat age, newest-harness-
    event age, elapsed vs timeout."
  - "AC-T2-2 — AnomalyRule(ABC) with evaluate(state)->Verdict|None;
    concrete ApproachingTimeout (elapsed > 0.8*task_timeout, not
    terminal), Stall (heartbeat fresh < crash_seconds BUT no new
    harness.* for > stall_seconds), Crashed (no heartbeat for >
    crash_seconds). Strategy set; thresholds from config/flags
    (G2 defaults: stall=60s, crash=120s, approaching=0.8)."
  - "AC-T2-3 — On first firing rule: prints a distinct ALERT line
    and exits non-zero (so it's usable as a proactive gate in an
    experiment harness); clean terminal state exits zero."
  - "AC-T2-4 — Read client reuses vfobs_sdk (a read helper) OR a
    thin typed client; auth via the WG2 vtf-token bearer; a read
    error degrades to a logged 'watch unavailable' tick, never a
    crash (symmetry with D-T0-1)."
  - "AC-T2-5 — Unit: tests/unit/test_watch_rules.py — each rule
    fires/!fires on crafted state tables incl. the exact pi-hang
    case (alive + no harness progress => Stall) and boundary
    (0.79 vs 0.81 timeout)."
  - "AC-T2-6 — Integration: tests/integration/test_watch_live.py —
    against a real vfobs with seeded events: a seeded stalled task
    yields STALLED, a near-timeout one APPROACHING_TIMEOUT, a
    healthy progressing one OK."
  - "AC-T2-7 — `vfobs-watch` is a console_script entry point in the
    vfobs_sdk (or a vfobs `tools/` module) — installable, not a
    loose script."
  - "AC-T2-8 — Branch wg-vfobs-ctrlinstr/t2-live-watcher pushed +
    PR open."

---

# Spec

## Files touched
- `sdk/python/vfobs_sdk/watch.py` (rules + poller + CLI) and the
  console_script wiring in `sdk/python/pyproject.toml`
  (`[project.scripts] vfobs-watch = "vfobs_sdk.watch:main"`).
- `sdk/python/vfobs_sdk/read_client.py` (thin typed read client
  over the WG2 API) if not already factored in T0.
- `tests/unit/test_watch_rules.py`,
  `tests/integration/test_watch_live.py`.

## Implementation sketch

**Patterns: Strategy (AnomalyRule set — the SAME rule objects WG4
later runs server-side) + a small poll loop holding WatchState.**

```python
@dataclass
class WatchState:
    claimed_at: datetime | None
    last_heartbeat_at: datetime | None
    max_harness_event_id: int | None
    terminal: bool
    now: datetime
    task_timeout_s: float

class AnomalyRule(ABC):
    @abstractmethod
    def evaluate(self, s: WatchState) -> Verdict | None: ...

class Stall(AnomalyRule):
    def __init__(self, stall_s, crash_s): ...
    def evaluate(self, s):
        if s.terminal or s.last_heartbeat_at is None: return None
        hb_age = (s.now - s.last_heartbeat_at).total_seconds()
        prog_age = ...  # since max_harness_event_id advanced
        if hb_age < self._crash and prog_age > self._stall:
            return Verdict("STALLED", "alive but no harness progress")
```

Poller builds WatchState from `/tasks/<id>/events` (heartbeat +
harness ids) each interval; runs rules in priority order; prints;
exits on first ALERT.

## Fail-loud directive (R3)

If the read client, a rule, or the live integration cannot be
completed: report explicitly — do not weaken AC-T2-5/6.

# Constraints
- T2 is INDEPENDENT of T0/T1 at build time (consumes the live WG2
  read API); it is tested against seeded events, not a live
  controller. Its end-to-end value is proven in T3.
- Rules MUST be pure functions of WatchState (no I/O) so WG4 can
  reuse them server-side unchanged.
- Watcher read failure never crashes the watch loop.

# Out of scope
- Auto-intervention (kill/retry) — surface only.
- Server-side anomaly emission (WG4). SSE (WG3).

# References
- plan.md §D4, §D7. workgraph.md WG-AC4.
- WG2 read-api endpoints (the data source).
- DESIGN §4 G2 (rule definitions + default thresholds).
