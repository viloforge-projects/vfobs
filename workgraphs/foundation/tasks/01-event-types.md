---
id: t_M7sV9w
slug: event-types
title: vfobs event types — base + 6 namespaces + Factory + JSON Schema export
workgraph: foundation
order: 1
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t1-event-types 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T1-1 — `src/vfobs/events/base.py` defines `Event` (Pydantic
    BaseModel), `EventNamespace` (Enum with values
    `WORKGRAPH, TASK, HARNESS, GATE, JUDGE, ANOMALY`), and
    `Classification` (Enum with values `PUBLIC, INTERNAL, SECRET`).
    Event base fields exactly match plan.md §D3: `v`, `type`,
    `workgraph_id`, `task_id` (Optional), `agent_id` (Optional),
    `trace_id` (Optional), `source`, `timestamp`, `classification`,
    `org_id`, `cluster_id`, `data` (subclass-specific).
  - AC-T1-2 — Six namespace modules in
    `src/vfobs/events/namespaces/` define concrete event classes:
    `workgraph.py` (WorkgraphCreated, WorkgraphStateChanged,
    WorkgraphCompleted); `task.py` (TaskClaimed, TaskStateChanged,
    TaskHeartbeat, TaskWorkdirChanged); `harness.py`
    (HarnessTurnStarted, HarnessToolCall, HarnessTurnCompleted);
    `gate.py` (GateStarted, GatePassed, GateFailed); `judge.py`
    (JudgeStarted, JudgeReviewSubmitted, JudgeDecision);
    `anomaly.py` (AnomalyStuckDetected). Each carries a typed
    `data: <NamespaceVerbData>` sub-model with the fields
    enumerated in the implementation sketch below.
  - AC-T1-3 — `src/vfobs/events/factory.py` defines `EventFactory`
    with one classmethod per event class
    (`EventFactory.task_state_changed(...)`,
    `EventFactory.harness_tool_call(...)`, etc.). Each method
    returns the typed event instance with `v=1`, `type` correctly
    set to the literal `"<namespace>.<verb>"`, and timestamp
    defaulted to `datetime.now(UTC)` if not supplied. This is the
    public construction surface; consumers MUST use the factory.
  - AC-T1-4 — `src/vfobs/events/schema.py` exports
    `dump_event_schemas() -> dict[str, dict]` returning a mapping
    `event_type → JSON Schema (Pydantic v2 model_json_schema())`.
    The output is stable JSON (sorted keys) suitable for committing
    to a contract-test fixture.
  - AC-T1-5 — Unit: `tests/unit/test_events.py` covers
    construction + validation of each event type via the
    factory: success paths (right fields → instance constructs),
    failure paths (wrong `type` literal → ValidationError; missing
    required field → ValidationError; bad `classification` value →
    ValidationError; bad `type` string not matching `^<ns>\.` →
    ValidationError).
  - AC-T1-6 — Contract: `tests/contract/test_event_schema.py`
    compares the output of `dump_event_schemas()` against a
    committed fixture at
    `tests/fixtures/event_schemas.v1.json`. Test FAILS on any
    diff. Rationale: schema-version v1 is a public contract per
    NFR1; any change is intentional and the fixture is updated
    alongside the model change (`pytest --update-fixtures` flow
    not built here; reviewer eyeballs the diff).
  - AC-T1-7 — Adding a new event type to namespace X follows the
    documented pattern (extend `<namespace>.py` with a new
    Pydantic class; add a factory method; update the v1 schema
    fixture). The unit test `test_add_new_event_type_follows_pattern`
    documents this with a synthetic `Task.test_synthetic` event
    constructed inline and asserts the schema dump includes it
    when injected. (Extensibility AC per engineering principles §5.2.)
  - AC-T1-8 — Branch `wg-vfobs-foundation/t1-event-types` is
    pushed to `viloforge/vfobs` and a PR is open.

---

# Spec

## Files touched

- `src/vfobs/events/__init__.py` (new) — public re-exports.
- `src/vfobs/events/base.py` (new) — `Event`, `EventNamespace`,
  `Classification` enums.
- `src/vfobs/events/factory.py` (new) — `EventFactory` classmethods.
- `src/vfobs/events/schema.py` (new) — `dump_event_schemas()`.
- `src/vfobs/events/namespaces/__init__.py` (new).
- `src/vfobs/events/namespaces/workgraph.py` (new).
- `src/vfobs/events/namespaces/task.py` (new).
- `src/vfobs/events/namespaces/harness.py` (new).
- `src/vfobs/events/namespaces/gate.py` (new).
- `src/vfobs/events/namespaces/judge.py` (new).
- `src/vfobs/events/namespaces/anomaly.py` (new).
- `tests/unit/test_events.py` (new).
- `tests/contract/test_event_schema.py` (new).
- `tests/fixtures/event_schemas.v1.json` (new) — JSON Schema
  baseline for the v1 contract.

(NOT touched: anything under `src/vfobs/repositories/`,
`src/vfobs/api/`, `src/vfobs/db.py`, or `alembic/`. T1 produces
pure-data Python types and their schema.)

## Implementation sketch

**Patterns: Factory + Specification (events as data).** The Factory
keeps construction in one place (consumers `from vfobs.events import
EventFactory`). Specification is the events-as-data discipline —
event classes carry no behavior, only fields and validators.

### base.py

```python
from datetime import datetime, UTC
from enum import Enum
from typing import Annotated, Literal
from pydantic import BaseModel, ConfigDict, Field

class EventNamespace(str, Enum):
    WORKGRAPH = "workgraph"
    TASK = "task"
    HARNESS = "harness"
    GATE = "gate"
    JUDGE = "judge"
    ANOMALY = "anomaly"

class Classification(str, Enum):
    PUBLIC = "public"
    INTERNAL = "internal"
    SECRET = "secret"

class Event(BaseModel):
    model_config = ConfigDict(frozen=True, str_strip_whitespace=True)

    v: int = 1
    type: str  # constrained by subclasses to a Literal
    workgraph_id: str = Field(min_length=1)
    task_id: str | None = None
    agent_id: str | None = None
    trace_id: str | None = None
    source: str = Field(min_length=1)
    timestamp: datetime
    classification: Classification = Classification.INTERNAL
    org_id: str = "viloforge"
    cluster_id: str = "vafi-dev"
```

The `data` field is added in each concrete subclass with its typed
sub-model.

### namespaces/task.py (representative; other namespaces follow the same shape)

```python
from typing import Literal
from pydantic import BaseModel
from vfobs.events.base import Event

class ExecutionSummary(BaseModel):
    num_turns: int | None = None
    total_tokens: int | None = None
    cost_usd: float | None = None

class TaskStateChangedData(BaseModel):
    from_status: str
    to_status: str
    execution_summary: ExecutionSummary | None = None

class TaskStateChanged(Event):
    type: Literal["task.state_changed"] = "task.state_changed"
    data: TaskStateChangedData

# Similar pattern for TaskClaimed, TaskHeartbeat, TaskWorkdirChanged.
```

Per-namespace event fields (data sub-model contents):

| Event | Data fields |
|---|---|
| `workgraph.created` | `created_by`, `kind`, `target_repos: list[str]` |
| `workgraph.state_changed` | `from_status`, `to_status` |
| `workgraph.completed` | `terminal_status`, `task_count` |
| `task.claimed` | `claimed_by_agent_id` |
| `task.state_changed` | `from_status`, `to_status`, `execution_summary?` |
| `task.heartbeat` | `at` (timestamp), `current_turn?: int` |
| `task.workdir_changed` | `files_changed: int`, `commits: int`, `branch?: str` |
| `harness.turn_started` | `turn_number`, `model`, `prompt_tokens?` |
| `harness.tool_call` | `turn_number`, `tool_name`, `tool_args_summary?` |
| `harness.turn_completed` | `turn_number`, `completion_tokens?`, `duration_ms?` |
| `gate.started` | `gate_name`, `command` |
| `gate.passed` | `gate_name`, `duration_ms?` |
| `gate.failed` | `gate_name`, `exit_code`, `stderr_tail?` |
| `judge.started` | `judge_agent_id` |
| `judge.review_submitted` | `per_ac_verdicts: dict[str, str]` |
| `judge.decision` | `decision: Literal["approved","changes_requested","rejected"]`, `notes?` |
| `anomaly.stuck_detected` | `last_activity_at: datetime`, `T_M_seconds: int`, `last_seen_event_type?: str` |

These are starting fields. Each carries `org_id`/`cluster_id` via the
`Event` base.

### factory.py

```python
from datetime import datetime, UTC
from vfobs.events.namespaces.task import TaskStateChanged, TaskStateChangedData
# ... imports for other namespaces

class EventFactory:
    @classmethod
    def task_state_changed(
        cls, *, workgraph_id: str, task_id: str, source: str,
        from_status: str, to_status: str,
        execution_summary: ExecutionSummary | None = None,
        timestamp: datetime | None = None,
        **base_kwargs,
    ) -> TaskStateChanged:
        return TaskStateChanged(
            workgraph_id=workgraph_id, task_id=task_id, source=source,
            timestamp=timestamp or datetime.now(UTC),
            data=TaskStateChangedData(
                from_status=from_status, to_status=to_status,
                execution_summary=execution_summary,
            ),
            **base_kwargs,
        )

    # ... one classmethod per event type
```

### schema.py

```python
import json
from vfobs.events.namespaces import workgraph, task, harness, gate, judge, anomaly

EVENT_CLASSES = [
    workgraph.WorkgraphCreated, workgraph.WorkgraphStateChanged,
    workgraph.WorkgraphCompleted, task.TaskClaimed,
    task.TaskStateChanged, task.TaskHeartbeat,
    task.TaskWorkdirChanged, harness.HarnessTurnStarted,
    harness.HarnessToolCall, harness.HarnessTurnCompleted,
    gate.GateStarted, gate.GatePassed, gate.GateFailed,
    judge.JudgeStarted, judge.JudgeReviewSubmitted,
    judge.JudgeDecision, anomaly.AnomalyStuckDetected,
]

def dump_event_schemas() -> dict[str, dict]:
    out = {}
    for cls in EVENT_CLASSES:
        # type literal -> class
        type_value = cls.model_fields["type"].default
        out[type_value] = cls.model_json_schema()
    return dict(sorted(out.items()))
```

The contract test `test_event_schema.py` does:

```python
def test_event_schemas_match_v1_fixture():
    current = dump_event_schemas()
    fixture_path = Path(__file__).parent.parent / "fixtures" / "event_schemas.v1.json"
    fixture = json.loads(fixture_path.read_text())
    assert current == fixture, "Event schemas drifted from v1 fixture. " \
        "Update the fixture only when intentionally bumping schema version."
```

## Fail-loud directive (R3)

If any required step (test creation, fixture generation, branch
push) cannot be completed due to missing dependencies, missing
tools, or blocked external constraints: report the failure
explicitly in your completion notes — do not rationalize partial
completion as success. The judge will fail tasks that report
dishonest success; tasks that report failure honestly can be
reworked.

# Constraints

- Event classes are **frozen** (immutable after construction).
  `Event.model_config = ConfigDict(frozen=True, ...)` enforces this.
  Rationale: events are append-only data; mutation after creation is
  a logic error.
- The `type` field on each concrete class is `Literal["<ns>.<verb>"]`
  with the correct default. This makes Pydantic auto-discriminate
  the union on POST /events (T4) without manual discriminator code.
- Schema version 1 is locked. Adding a field is a schema-version
  bump (out of scope for WG1). Breaking changes require coordinated
  consumer + producer SDK version bumps — that discipline is part
  of NFR1.
- `org_id` and `cluster_id` carry defaults — controllers do not
  need to populate them in v1. They MUST NOT be removed or renamed
  in v1; v2 multi-tenant work depends on these hooks being present
  from the start.

# Out of scope

- Event serialization optimization (msgpack, protobuf). v1 is JSON
  via Pydantic.
- Event ID assignment (BIGSERIAL is server-side; T1's events have
  no `id` field — the repository in T2 returns the assigned id).
- Real-time event publishing / pub-sub. WG3 work.
- Anomaly detection logic. WG4 work — T1 only defines the
  `anomaly.stuck_detected` event *type*, not when it fires.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §C
- Pydantic v2 docs (discriminated unions via Literal):
  https://docs.pydantic.dev/latest/concepts/unions/#discriminated-unions
- `vtaskforge/vtf-sdk-python/vtf_sdk/` — reference for Pydantic-typed
  payload patterns
