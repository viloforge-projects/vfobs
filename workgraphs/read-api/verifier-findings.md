---
authored_by: claude-opus-4-7
created_at: 2026-05-16T00:00:00Z
methodology: vtf-methodologies/verifier/bugfix.md (V1-V15)
target: workgraphs/read-api/ at commit before inline patches
verdict: ready-after-inline-patches
---

# Verifier findings — WG2 read-api

Pass run by Claude in the same session, ahead of vtf import. Bias
toward strict reading per methodology. **Two BLOCKING defects found
and patched inline** (both the WG1-F1 class: spec assumes a
field-flow the data layer does not carry / artifacts disagree on a
correctness-bearing filter). Four advisory findings recorded; two
patched inline, two deferred to retro.

## Check summary

| Check | Result | Note |
|---|---|---|
| V1 (no `[NEEDS CLARIFICATION]` / `TBD`) | PASS | grep clean across workgraph.md + plan.md + 6 tasks |
| V2 (DAG acyclic + connected) | PASS | T0,T1 roots; T5 sink; T0/T1→T2/T3→T4→T5; all `depends_on` ids resolve; matches workgraph.md mermaid |
| V3 (target_repo resolves) | PASS | viloforge/vfobs exists (WG1 shipped there) |
| V4 (AC verifiability) | PASS w/ advisory F-adv1 | All ACs observer-checkable; AC-T2-2 had a misleading parenthetical (patched) |
| V5 (externally-grounded AC) | PASS | Every task has a "branch pushed" AC; T0 also a "PR open" AC |
| V6 (test_command populated) | PASS | 6/6 tasks carry the `git ls-remote --heads` gate |
| V7 (fail-loud directive) | PASS | Verbatim R3 fail-loud block in every task; T5 carries the WG1-T6 "not done unless scenario green" intensifier |
| V8 (file:line citations resolve) | PASS w/ **BLOCKING F1** | All cited paths exist; column list matches DDL exactly; but `_row_to_event`/`Event` do not carry `id` — see F1 |
| V9 (mock/LSP idioms) | PASS | `@pytest.fixture(params=["inmemory","postgres"])` `repo_under_test` is the real WG1 LSP pattern; T1 AC-T1-8 matches. Greenfield VtfClient double — N/A like WG1 |
| V10 (cross-task/artifact AC consistency) | **BLOCKING F2** + advisory F-adv1 | plan §D5 (filter on terminal `to_status`) vs AC-T1-6/SQL (filter on `execution_summary` presence only) vs WG-AC5 ("every event") — three-way disagreement on a correctness-bearing filter. F-adv1: AC-T2-2 implied a `find_by_workgraph(order=desc)` param T1 does not expose |
| V11 (methodology pinning matches kind) | PASS w/ advisory F-adv3 | kind=infrastructure; vtf-methodologies has only `bugfix.md` — same bootstrap-period gap as WG1-F5 |
| V13 (test ACs at every pyramid level) | PASS | T0 unit+int; T1 unit+int (LSP); T2/T3/T4 unit+int+contract; T5 scenario. Levels match scope |
| V14 (pattern claims coherent) | PASS | ReadAuth Strategy = real ABC + 2 impls + DI site (mirrors WG1 IngestAuth). **T0 "Adapter" is genuine** (projects vtaskforge HTTP/JSON → typed vfobs domain) — NOT the WG1-F4 theater class; spec-author applied the V14 lesson. Builder (EventQuery), Repository-extension, cost Strategy all coherent |
| V15 (extensibility affordances) | PASS | `org_id` on EventQuery (default viloforge); cost-aggregation kind is Strategy; ReadAuth admits OIDC v2 |

## Blocking defects (patched inline)

### F1 — `Event` carries no `id`; WG2 pagination/`last_event_id` build on `events[-1].id`

**Severity:** blocking (read-side ghost-field; the WG1-F1 class).

**Root cause.** `src/vfobs/events/base.py` `Event` (frozen) declares
no `id`. `event_repository.py:_row_to_event` reconstructs an `Event`
from a row but maps neither `id` nor `created_at`; `InMemory.store`
returns a separate `eid` int and stores the bare `Event`. WG1 only
needed `store()→int` + `get_by_id(id)`, so `Event.id` was never
required. WG2 changes that:

- plan §D4 + AC-T2-4 + T3 sketch: `next_from_id = events[-1].id + 1`
- AC-T2-2: `vfobs.last_event_id`
- AC-T2-5: `EventDTO is Event.model_dump'able — no separate transform`

`events[-1].id` raises `AttributeError`; and if `id` were attached
as an *undeclared* key, `model_dump()` (the EventDTO path) silently
drops it.

**Empirically verified** (frozen-model repro, this session):

```
undeclared id via model_copy  -> getattr=42  but  "id" in model_dump() = False   # silent drop (WG1-F1 class)
declared id: int|None = None  -> model_copy(update={"id":99}); model_dump()["id"] = 99; original stays None
```

**Inline patch applied:**

- `Event` base gains a declared `id: int | None = None`
  (frozen-safe, R13-safe — declared, so it survives `model_dump`).
- `_row_to_event` maps `row["id"]`; the WG1 `store→int` /
  `get_by_id` contract is unchanged (write-side `id` stays `None`;
  DB/counter assigns; reads reconstruct with `id`).
- T1 (task 01): `Files touched` adds `src/vfobs/events/base.py`;
  `find_*` Postgres impls already SELECT `id` → reconstruct it;
  InMemory `find_*` attach via `model_copy(update={"id": eid})`;
  new AC-T1-10 asserts `id` populated on every `find_*` result AND
  present in `model_dump()` (regression guard for the WG1-F1 class).
- T2 cross-refs F1 for `last_event_id` / `next_from_id`.
- plan §D3 notes the `Event.id` propagation contract.

**⚠️ Mechanism superseded at implementation (operator decision R2,
2026-05-16).** The inline-patch mechanism above ("add a declared
`id` to the base `Event`") was found, during T1 implementation, to
drift WG1's deliberately-locked `tests/fixtures/event_schemas.v1.json`
— adding `id` to the base model changes the **write/ingest** schema
of every event type (a client could POST `{"id": …}`), and the
contract suite went red. F1's *finding* (pagination needs a stored
id) stands; its *mechanism* was wrong. Operator chose **R2**:

- `Event` base model is **unchanged** — the locked v1 ingest schema
  stays locked (`contract` suite stays green). `_row_to_event` and
  `get_by_id` are byte-for-byte unchanged.
- A new read-only model `repositories.StoredEvent { id: int;
  event: Event }` carries the DB id. `find_*` return
  `list[StoredEvent]`. Read endpoints serialize `{id, event}`.
- AC-T1-10 reworded to the StoredEvent mechanism; plan §D3/§D4 and
  tasks 01/02/03 re-patched accordingly. The "no separate transform
  layer" intent of AC-T2-5 is consciously reversed — a thin
  StoredEvent read model is the correct write/read separation and
  is cheaper than rewriting a locked contract.
- Methodology: this is a new verifier-class — *"an inline
  field-add patch to a model whose serialized schema is a locked
  contract."* The verifier must check whether a model it proposes
  to mutate backs a locked contract fixture. Recorded for WG2 retro
  alongside the D-T0-1 class.

### F2 — Cost-filter disagreement (plan §D5 ↔ AC-T1-6 ↔ WG-AC5) + rework double-count risk

**Severity:** blocking (correctness of WG-AC5 cost rollups).

**Root cause.** Three artifacts prescribe different cost-aggregation
filters:

- plan §D5: `task.state_changed` AND `execution_summary` not null
  AND `data.to_status` is terminal.
- AC-T1-6 + `_COST_BY_WORKGRAPH` sketch: `task.state_changed` AND
  `execution_summary` present (no terminal predicate).
- WG-AC5: "sums **every** `task.state_changed` event with
  `data.execution_summary.cost_usd` set".

DESIGN G8 (pipeline-observability-DESIGN.md:192-200) says
`execution_summary` is produced "on **task-completion events**".
Under judge-driven rework a task can complete an executor run more
than once (doing→changes_requested→doing→done); WG5 instrumentation
(which actually emits these events) does not exist yet, so the
per-task emission cardinality is **not guaranteed to be 1**.
"Sum every event" (WG-AC5) then double-counts cost across rework
cycles. plan §D5's terminal filter narrows it but still admits
multiple terminal-ish transitions and forces vfobs to hardcode the
vtaskforge state-machine's terminal set (a coupling + a latent
`[NEEDS CLARIFICATION]`).

**Inline patch applied — de-dup, state-machine-agnostic:** cost
aggregation counts **one `execution_summary` per `task_id`: the one
on the highest-`id` `task.state_changed` event that carries an
`execution_summary`**. Correct whether rework emits 1 or N summary
events; needs no vtaskforge terminal-status table; matches WG-AC5's
intent ("cost of the workgraph", not "sum of all rework attempts").

- plan §D5 rewritten to the de-dup discipline + this rationale.
- AC-T1-6 rewritten: "latest `execution_summary` per `task_id`";
  `_COST_*` SQL → `DISTINCT ON (task_id) ... ORDER BY task_id, id
  DESC` subquery then aggregate; InMemory → keep highest-`eid` per
  `task_id`. `task_count` = distinct contributing `task_id`;
  `sample_event_count` = contributing (post-dedup) event count.
- workgraph.md WG-AC5 wording aligned ("latest per task", no NaN).
- MQ3 (plan open question) updated: answered by F2.

## Advisory findings

- **F-adv1 (patched inline).** AC-T2-2 said
  `find_by_workgraph(limit=1, order=desc-or-via-from_id-trick)`,
  but T1's `find_by_workgraph` exposes no `order` param (plan §D3
  fixes `id ASC`); the T2 spec body itself resolves this with a
  T2-internal private `_find_last_by_workgraph`. AC-T2-2 reworded
  to reference that helper; no interface pollution.
- **F-adv2 (retro).** Two `Principal` types now exist —
  `api/auth.py` (write) and `api/read_auth.py` (read). Acceptable
  for v1 (independent strategies, D6 "two strategies coexist"); a
  shared `Principal` is a v2 consolidation. Recorded for retro.
- **F-adv3 (retro).** V11 bootstrap gap: workgraph is
  `kind: infrastructure` but only `*/bugfix.md` methodologies
  exist. Same finding as WG1-F5; informational per V11 bootstrap
  clause until an `infrastructure.md` methodology lands.
- **F-adv4 (patched inline).** T0's `main.py` lifespan sketch used
  `if not getattr(app.state, "vtf_client", None)`; existing
  `main.py:20-23` uses `if not hasattr(...) or ... is None`.
  Sketch aligned to the existing idiom (V9 spirit — match the
  scaffold).

## Verdict

`specced → ready-after-inline-patches`. F1 + F2 patched inline with
empirical/authoritative grounding; F-adv1/F-adv4 patched inline;
F-adv2/F-adv3 carried to WG2 retro. Workgraph is ready for vtf
import and executor pickup.

## References

- `vtf-methodologies/verifier/bugfix.md` V1-V15
- `workgraphs/foundation/verifier-findings.md` — F1/F4 precedent
  this pass extends (same silent-field-drop + pattern-theater
  classes)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` G8 —
  execution_summary emission authority used to ground F2

---

# Post-hoc finding: OIQ3 external contract was INVENTED (this pass missed it)

**Recorded 2026-05-16, after the real vafi-dev end-to-end.
Preserved, not rewritten — the miss is the lesson.**

This verifier pass passed D1/D2 (it even praised the Adapter as
"genuine" under V14). It should not have: D1's `GET
/v2/auth/whoami` + `Bearer` and D2's `/v2/workgraphs/<id>/` are
**invented — none exist in real vtaskforge** (it has
`/v2/auth/validate/` + DRF `Token`; "workgraph" == milestone).

Why every gate missed it: V1–V15 only resolve **in-repo**
citations (V8). Nothing required the *external* contract to be
cited against the real provider, and the WG2 stub faked the
invented endpoint — so the verifier, 6 PRs, and the gated
scenario were all internally consistent and externally wrong.
It surfaced only when a real vfobs hit real vtaskforge.

Root cause + rule: the architect resolved an under-specified
external contract by inventing a plausible endpoint, then a
self-authored stub validated the invention against itself (the
closed-system rubber-stamp at an external boundary). This
produced **verifier V18** (external-integration contracts must
cite the real provider source/OpenAPI; stubs derived from it,
never invented) and kb `feedback-external-contract-grounding`.
Corrected in vfobs#20. Current truth: `viloforge/vfobs`
`docs/IMPLEMENTATION-STATUS.md`.
