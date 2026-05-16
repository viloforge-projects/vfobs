---
id: t_ci_sdk0
slug: vfobs-sdk-emitter
title: vfobs-sdk-python — versioned fire-and-forget Emitter
workgraph: controller-instrumentation
order: 0
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-ctrlinstr/t0-vfobs-sdk-emitter 2>&1 | grep -q .
created_at: 2026-05-16T00:00:00Z
acceptance_criteria:
  - "AC-T0-1 — sdk/python/ is a buildable package `vfobs_sdk`
    (pyproject mirrors vafi/vtf-sdk-python: setuptools, pydantic>=2,
    httpx; exposes `vfobs_sdk.__version__` semver string)."
  - "AC-T0-2 — Emitter(ABC) with `emit(event) -> None`. Concrete
    HttpEmitter(url, token, *, timeout=2.0, queue_max=1000) +
    NullEmitter (no-op) + a test BufferingEmitter (records events).
    Strategy pattern; controller depends on the ABC."
  - "AC-T0-3 — HttpEmitter.emit NEVER raises: network/4xx/5xx/
    timeout/serialization errors are caught; emit() is O(1) — it
    enqueues onto a bounded queue (maxlen=queue_max, drop-oldest on
    overflow) drained by ONE background task that POSTs /events."
  - "AC-T0-4 — per-emit hard timeout (default 2.0s) on the HTTP
    call; a hung endpoint cannot block the drain task beyond it."
  - "AC-T0-5 — first delivery failure logs once at WARNING;
    subsequent failures are rate-limited (no unbounded log spam)."
  - "AC-T0-6 — `make_emitter(settings) -> Emitter` factory: returns
    NullEmitter unless emit is enabled+configured (URL+token).
    org_id/cluster_id passthrough supported."
  - "AC-T0-7 — Unit: tests/unit/test_emitter.py — emit never raises
    on (a) connection refused (b) 503 (c) timeout (d) bad payload;
    overflow drops oldest; NullEmitter is a true no-op; factory
    selects Null when unconfigured."
  - "AC-T0-8 — Integration: tests/integration/test_emitter_live.py
    — HttpEmitter against a real vfobs (ASGITransport) round-trips
    a task.heartbeat + a task.state_changed; on vfobs 5xx the
    caller still returns promptly and does not raise."
  - "AC-T0-9 — Contract: tests/contract/test_emitter_envelope.py —
    the JSON the SDK emits for each event type validates against
    WG1 tests/fixtures/event_schemas.v1.json (no ingest-schema
    drift; V17 discipline)."
  - "AC-T0-10 — Branch wg-vfobs-ctrlinstr/t0-vfobs-sdk-emitter
    pushed to viloforge/vfobs + PR open."

---

# Spec

## Files touched (all NEW unless noted)
- `sdk/python/pyproject.toml`, `sdk/python/vfobs_sdk/__init__.py`
  (`__version__`), `sdk/python/vfobs_sdk/emitter.py`,
  `sdk/python/vfobs_sdk/events.py` (re-export/vendor the WG1 event
  models the SDK serializes — see Constraints).
- `tests/unit/test_emitter.py`,
  `tests/integration/test_emitter_live.py`,
  `tests/contract/test_emitter_envelope.py`.
- (NOT touched: server `src/vfobs/**`, WG1/WG2 tests, charts.)

## Implementation sketch

**Patterns: Strategy (Emitter ABC → Http/Null/Buffering) + a
bounded producer/consumer queue for non-blocking delivery.**

```python
class Emitter(ABC):
    @abstractmethod
    def emit(self, event: Event) -> None: ...
    async def aclose(self) -> None: ...

class NullEmitter(Emitter):
    def emit(self, event): pass

class HttpEmitter(Emitter):
    def __init__(self, url, token, *, timeout=2.0, queue_max=1000):
        self._q = collections.deque(maxlen=queue_max)  # drop-oldest
        self._task = asyncio.create_task(self._drain())
        ...
    def emit(self, event):                 # O(1), never raises
        try: self._q.append(event)
        except Exception: pass
    async def _drain(self):
        while True:
            ev = await self._next()
            try:
                await self._http.post("/events", json=ev.model_dump(mode="json"),
                                      headers=..., timeout=self._timeout)
            except Exception as e:
                self._log_rate_limited(e)  # never propagates
```

## Fail-loud directive (R3)

If any required step (package build, real-vfobs integration,
contract fixture) cannot be completed: report the failure
explicitly in completion notes — do not rationalize partial
completion as success.

# Constraints
- The SDK MUST NOT add a runtime dependency that drifts the WG1
  ingest schema; it serializes the SAME event envelope WG1 accepts
  (AC-T0-9 pins this). Vendoring vs importing the WG1 models is an
  impl choice — whichever, the contract test is authoritative.
- emit() is sync + O(1); all I/O is in the background drain.
- No global singletons — the Emitter is constructed and injected.

# Out of scope
- The controller hook calls (T1). Server-side anomaly (WG4).
  SSE (WG3). npm/TS SDK (later).

# References
- plan.md §D1, §D3, §D5. workgraph.md.
- `vafi/vtf-sdk-python/` packaging precedent.
- WG1 `tests/fixtures/event_schemas.v1.json` (contract target).
- `vtf-methodologies/executor/bugfix.md`.
