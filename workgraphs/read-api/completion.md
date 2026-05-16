---
authored_by: claude-opus-4-7
created_at: 2026-05-16T00:00:00Z
workgraph: read-api
status: complete
merge_commit: b1ecf2f
---

# WG2 read-api — completion

The read side of vfobs is live on `viloforge/vfobs:main`
(@ `b1ecf2f`). All six tasks implemented under SDD discipline by
Claude in operator chat (vafi-executor still maturing), each its
own branch + PR, full TDD pyramid, squash-merged sequentially.

## Workgraph acceptance criteria

| AC | Status | Evidence |
|----|--------|----------|
| WG-AC1 — all 6 task PRs merged to viloforge/vfobs:main | ✅ | #10 #11 #12 #13 #14 #15 |
| WG-AC2 — unit+integration+contract pass on merge commit | ✅ | `b1ecf2f`: 82 unit / 3 contract / 38 integration |
| WG-AC3 — T5 scenario passes e2e on kind | ✅ | full scenario suite 6/6 green on kind (WG1 3 + WG2 3) |
| WG-AC4 — vtf-token piggyback (valid→authn, invalid/unauth→401) | ✅ | T0 unit (cache/401), T2/T3/T4/T5 401 tests, scenario `test_read_endpoints_require_auth` |
| WG-AC5 — cost rollup = latest execution_summary per task, NULL-tolerant | ✅ | F2 de-dup; T1/T4 unit+integration, T5 scenario assert reworked task counted once, no NaN |
| WG-AC6 — vtaskforge project shows 6 WG2 tasks done | ✅ | all 6 reset→done (operator-implemented; engineering verified independently) |

## What shipped

- **T0** (#10) — `VtfClient` Adapter + `ReadAuth` Strategy
  (`VtfTokenAuth` 60s sha256-prefix TTL cache + `StaticPrincipalAuth`).
- **T1** (#11) — `EventRepository` read methods
  (`find_by_workgraph/find_by_task/find_filtered/cost_summary`) +
  the `StoredEvent` read model (F1/R2) + F2 cost de-dup.
- **T2** (#12) — `GET /workgraphs/<id>`, `/tasks/<id>`,
  `/tasks/<id>/events`; tail/count repo helpers (D-T2-1).
- **T3** (#13) — `GET /events?filter=...` with the `EventQuery`
  Builder (unknown param / bad bounds → 422).
- **T4** (#14) — `GET /workgraphs/<id>/cost`, `/agents/<id>/cost`;
  `CostAggregationStrategy` (ByWorkgraph/ByAgent).
- **T5** (#15) — kind scenario + vtfstub sidecar; fail-loud
  verified end-to-end.

## Decisions ratified (operator)

- **D-T0-1** — vfobs read-side is *degradable*, not boot-critical
  (Option A). kb decision `jakdRrCh`.
- **R2** — verifier-F1's finding stood but its mechanism (declared
  `id` on base `Event`) drifted the locked ingest schema; replaced
  by the read-only `StoredEvent` model. Code + specs +
  verifier-findings re-patched.
- **D-T1-1** — `get_by_id` stays a bare `Event` (LSP + equality
  preserved); only `find_*` wrap `StoredEvent`.
- **D-T2-1** — tail/count helpers belong on the ABC (LSP-tested ⇒
  contract), reconciling the spec's "private/four" wording.

## Open items

None blocking. Methodology propagation items in `executor-retro.md`.
