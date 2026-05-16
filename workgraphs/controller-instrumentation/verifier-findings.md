---
authored_by: claude-opus-4-7
created_at: 2026-05-16T00:00:00Z
methodology: vtf-methodologies/verifier/bugfix.md (V1-V17)
target: workgraphs/controller-instrumentation/ pre-implementation
verdict: ready (advisories only; F-adv1 patched inline)
---

# Verifier findings — WG5-minimal (controller-instrumentation)

Authored same session as the design, with the WG1/WG2 retro
lessons fresh — so the design preemptively satisfies the new
V16/V17 and the fail-safe (D-T0-1) lineage. **No blocking
defects.** Three advisories; one patched inline.

## Check summary

| Check | Result | Note |
|---|---|---|
| V1 no NEEDS CLARIFICATION/TBD | PASS | grep clean |
| V2 DAG acyclic+connected | PASS | roots T0,T2 → T1 → T3 sink; depends_on ids resolve. F-adv1: mermaid had a redundant T0→T3 edge (transitive via T1) — trimmed inline |
| V3 target_repo resolves | PASS | viloforge/vfobs + vilosource/vafi both present (multi-repo wg) |
| V4 AC verifiability | PASS | every AC observer-checkable; fail-safe ACs (T1-6/T3-5) phrased as testable assertions |
| V5 externally-grounded AC | PASS | 4/4 branch-pushed+PR ACs |
| V6 test_command populated | PASS | 4/4 git ls-remote gates |
| V7 fail-loud R3 | PASS | 4/4; T3 carries the WG1-T6/WG2-T5 intensifier |
| V8 file:line citations | PASS | hook surfaces (controller.py _poll_and_execute claim/terminal, heartbeat.py heartbeat_loop, invoker.py invoke/_run_harness/_parse_harness_output), vfobs EventFactory (task_claimed/heartbeat/state_changed, harness_turn_*), vtf-sdk-python precedent — all verified against source this session |
| V9 mock idioms | PASS | BufferingEmitter/Null are real LSP substitutes (not mocks); matches the vfobs WG1/WG2 in-memory-double convention |
| V10 cross-task consistency | PASS | T0 defines the Emitter contract T1 consumes + the envelope T0/T3 assert; T2 independent (consumes live WG2 API) — no conflicting prescriptions |
| V11 methodology pinning | PASS (advisory F-adv3) | kind=infrastructure; only bugfix.md methodologies — bootstrap-period informational (WG1-F5/WG2 lineage) |
| V13 pyramid ACs | PASS | T0 unit+int+contract; T1 unit+int (envelope contract is T0's); T2 unit+int; T3 scenario — appropriate to each scope |
| V14 pattern coherence | PASS | Strategy (Emitter ABC + Http/Null/Buffering + factory; AnomalyRule set), DIP (controller↔Emitter ABC), bounded producer/consumer queue — all structurally real, no pattern theater |
| V15 extensibility | PASS | org_id/cluster_id passthrough; thresholds as config; SDK semver; AnomalyRule objects explicitly designed for WG4 server-side reuse |
| V16 required-config vs fixtures (NEW, WG2) | PASS | T1 adds VFOBS_EMIT_* but plan §D6 makes them default-off/optional → NullEmitter; no required-no-default field → no fixture regression. The D-T0-1 lesson applied preventively |
| V17 inline-patch vs locked contract (NEW, WG2) | PASS (advisory F-adv2) | T0 AC-T0-9 pins the emitted envelope against WG1 locked event_schemas.v1.json. Drift-prevention designed in. F-adv2: prefer importing the WG1 models over vendoring a copy |

## Advisories

- **F-adv1 (patched inline).** workgraph.md mermaid drew
  `T0 --> T3`; T3's `depends_on` is `[t_ci_hook1, t_ci_watch2]`
  (T0 reached transitively via T1). `depends_on` is authoritative;
  the extra mermaid edge was cosmetic-redundant. Trimmed so the
  diagram matches the DAG.
- **F-adv2 (retro/impl note).** T0 should *import* the WG1 event
  models, not vendor a copy — structurally prevents envelope
  drift; AC-T0-9's contract test is the backstop either way.
  Captured for the T0 executor + WG5 retro.
- **F-adv3 (retro).** V11 bootstrap gap (infrastructure kind,
  bugfix.md only) — informational; same lineage as WG1-F5 /
  WG2-F-adv3.

## Verdict

`specced → ready`. No blocking defects; F-adv1 patched inline;
F-adv2/F-adv3 carried to the WG5 retro. The workgraph is ready
for vtf import + implementation.

**Note for the operator:** unlike WG1/WG2 (greenfield), T1
modifies the PRODUCTION vafi controller loop. The design makes
emission default-off + fail-safe + AC-gated, but the
design→implementation boundary on a production component is a
legitimate review point (surfaced rather than auto-proceeded).

## References
- `vtf-methodologies/verifier/bugfix.md` V1-V17 (V16/V17 are the
  WG2-propagated checks this pass exercises first).
- WG2 `verifier-findings.md` (F1/F2/V16/V17 lineage),
  kb `jakdRrCh` (degradable → fail-safe basis).
