---
authored_by: claude-opus-4-7
created_at: 2026-05-14T00:00:00Z
methodology: vtf-methodologies/verifier/bugfix.md (R1-R15)
target: workgraphs/foundation/ at commit before inline patches
verdict: ready-after-inline-patches
---

# Verifier findings — WG1 foundation

Pass run by Claude in the same session that authored the artifacts.
Bias toward strict reading. One BLOCKING defect found and patched
inline; six advisory findings recorded for retro.

## Check summary

| Check | Result | Note |
|---|---|---|
| V1 (no `[NEEDS CLARIFICATION]` / `TBD`) | PASS | grep clean |
| V2 (DAG acyclic + connected) | PASS | T0/T1/T3 roots; T6 sink; no cycles |
| V3 (target_repo resolves) | PASS | viloforge/vfobs created in WG0 |
| V4 (AC verifiability) | PASS | All ACs are external-observer-checkable |
| V5 (externally-grounded AC) | PASS | Every task has "branch pushed" AC |
| V6 (test_command populated) | PASS | Every task has the git ls-remote gate |
| V7 (fail-loud directive) | PASS | Verbatim wording in every task |
| V8 (file:line citations resolve) | PASS w/ advisory F7 | Directory-level cites verified; few literal line cites because vfobs is greenfield |
| V9 (mock-double idioms) | N/A | No pre-existing test scaffold to mirror — greenfield repo |
| V10 (cross-task AC consistency) | **BLOCKING F1 — patched inline** | T4 Enricher's `server_received_at` would silently drop at Pydantic `model_dump`; patched to no-op Enricher + use `created_at` instead. Advisory F2/F3 also raised. |
| V11 (methodology pinning matches kind) | PASS w/ advisory F5 | workgraph is `kind: infrastructure`; vtf-methodologies has `bugfix.md` only — bootstrap-period gap |
| V13 (test ACs at every applicable pyramid level) | PASS w/ advisory F6 | Every code-bearing task hits its applicable levels; T3 could add a tiny contract AC for /healthz JSON shape but value is low |
| V14 (pattern claims coherent with sketch) | **F4 patched inline** | T0's "Adapter" claim was pattern theater; dropped. Other pattern claims (Repository, Factory, Strategy, CoR, Singleton, Decorator) verified coherent |
| V15 (extensibility affordances) | PASS | `org_id`, `cluster_id` present in T0 schema + T1 base Event; Strategy interfaces for auth (T4) admit per-pod / mTLS in v2 |

## Blocking defect (patched inline)

### F1 — Enricher's `server_received_at` would silently drop (T0/T4/T6)

**Severity:** blocking (ghost-write defect; affects a field needed
for anomaly-detection ordering).

**Root cause:** T4's spec body had:

```python
async def enricher(event, context):
    data = event.data.model_copy(update={
        "server_received_at": context.get("server_received_at",
                                          datetime.now(UTC)),
    })
    return event.model_copy(update={"data": data})
```

But the per-event-type `data` sub-models (T1) do not declare
`server_received_at`. Pydantic v2's `model_copy(update={...})`
*does* store the extra key on `__dict__` — but `model_dump()` (used
by T2's `_event_to_row` to serialize for Postgres INSERT) only
outputs declared fields. The field would be silently lost between
Enricher and storage.

**Empirically verified** with a 6-line Pydantic script:

```python
class Data(BaseModel):
    a: int

d = Data(a=1)
d2 = d.model_copy(update={"b": 2})
print(d2.model_dump())     # {'a': 1}  — 'b' lost
print(hasattr(d2, 'b'))    # True       — but only in __dict__
```

**Inline patch applied:**

1. **T4 Enricher** rewritten as a v1 no-op pass-through (CoR
   pipeline shape preserved as an extension point for v2 Python-
   side enrichments). AC-T4-4 reworded accordingly.
2. **Server-time audit** moved to the storage layer — Postgres
   `events.created_at` column (already in T0's schema with DEFAULT
   `now()`) is the canonical "server received" timestamp.
3. **T4 unit + integration ACs** updated to assert
   `created_at` is recent rather than checking the (gone)
   `server_received_at`.
4. **T6 scenario AC + sketch** updated to read `created_at` from
   the row and assert within-5s-of-now.
5. **Note on server-time audit** added to T4 spec body explaining
   the design choice + the empirical reason `data` smuggling is
   forbidden — defensively documented so a future spec doesn't
   regress.
6. **Constraint added to T4**: "any v2 Python-side enrichments add
   explicit base fields on `Event` (T1) + columns in T0; never
   undeclared keys."

**Rationale for patching at verify time** (vs returning to
spec-author): the defect is mechanical, the fix surface is tight
(3 tasks), and patches are pure spec text. Per verifier
methodology outcome 2: "Small defects can be patched inline at
verify time and noted." Recorded here for retro.

## F4 — Pattern theater on T0 (patched inline)

T0's body claimed Adapter pattern for Alembic ("Alembic adapts our
Python schema-definition to whatever Postgres version the cluster
runs"). Alembic is a migration tool, not a GoF Adapter — the claim
was decorative. Per engineering-principles §3 ("avoid pattern
theater"), patched T0 body to state no application-layer patterns
apply.

## Advisory findings (not blocking; record for retro)

### F2 — Scenario role posture: `vfobs_app` is both migrator and runtime user (T6)

In `scripts/scenario/prepare.sh`, alembic runs as `vfobs_app`
(which has DB-owner-like privileges via bitnami-created user), then
the vfobs pod also runs as `vfobs_app` (locked-down per T0's
grants). The runtime grants assertion (AC-T0-6 — "exactly SELECT
+ INSERT") may technically pass because
`information_schema.role_table_grants` reflects explicit grants,
not ownership-implicit privileges. But the security posture is
muddled.

Recommend (v2 or hardening pass): separate `vfobs_migrator` role
(creates schema + grants) from `vfobs_app` role (SELECT+INSERT
only). Out-of-scope-for-WG1 but flag for the workgraph retro and
production-deployment runbook.

### F3 — `VFOBS_APP_DB_PASSWORD` env var not declared in T3 Settings (T0)

T0 says alembic's `env.py` reads `VFOBS_APP_DB_PASSWORD` for the
role-creation parameter. T3's `Settings` doesn't include this
field — by design, because alembic env.py reads it directly. But
this means there are two env-var resolution paths for vfobs
secrets:

- Runtime app: `VFOBS_DATABASE_URL` (parsed by pydantic-settings)
- Migration: `VFOBS_APP_DB_PASSWORD` (read directly in alembic
  env.py)

Document this in T5's chart (the values for both Secrets are
ESO-provisioned) and in `docs/scenario-test-runbook.md`. Flag for
the workgraph retro.

### F5 — Methodology pinning kind-mismatch (V11; bootstrap-period)

`project.yaml` pins `vtf-methodologies@v0.0`. `workgraph.md`
declares `kind: infrastructure`. The methodology dir has only
`bugfix.md` files — no `infrastructure.md` files exist yet. The
plan's IMPLEMENTATION-PLAN §3 explicitly names these WGs as
`kind: infrastructure`, but the closest applicable methodology is
the bugfix one.

V11 says: "(During the bootstrap period — pre-methodology-pinning
— this check is informational only.)" So flagging only.

Recommend: when WG1's retro runs, draft
`vtf-methodologies/{spec-author,executor,verifier,judge}/
infrastructure.md` files seeded from this workgraph's experience
— mirrors the bugfix.md propagation we did from
vafi-rolling-restart-fix.

### F6 — T3 contract ACs for health-endpoint JSON shapes (advisory)

T3 has unit + integration ACs covering `/healthz`, `/readyz`,
`/metrics`. A contract-level AC pinning the JSON response shape
(`{"status": "ok"}`, `{"status": "ok", "db": "ok"}`, etc.) could
be added — they're public APIs that k8s probes depend on. Value is
low because the shapes are trivial and the integration test covers
them; not worth blocking.

### F7 — Most file references are directory-level, not file:line (R4 nuance)

Spec-author R4 mandates `file:line` citations for existing-code
references. Most of WG1's spec references point at directories or
whole files (`vtaskforge/vtf-sdk-python/vtf_sdk/`,
`vtaskforge/charts/vtf/`) because vfobs is greenfield — references
are pattern-mirror, not specific code points. Acceptable for WG1.

Note for future workgraphs: WG2-5 (which extend vfobs's existing
code) MUST follow R4 strictly with literal `file:line` cites.

## Outcome

**WG1 status:** `ready-after-inline-patches`. All blocking defects
patched. Six advisory findings recorded for the workgraph retro.

The PR (https://github.com/viloforge-projects/vfobs/pull/1) is
mergeable after a force-push that picks up the inline patches.

## Methodology-relevant cross-cuttings (candidates for retro)

- **VX1.** The Pydantic `model_copy` × `model_dump` interaction
  (F1) is a class of defect that could repeat anywhere a spec
  mutates an event/typed-model via `update={...}`. Recommend a new
  spec-author rule: "When mutating a typed Pydantic model, only
  use keys declared on the target class; verify with a model_dump
  round-trip in the sketch." Candidate **spec-author R13**.
- **VX2.** Bootstrap-period methodology-kind mismatch (F5) is the
  first concrete instance triggering an `infrastructure.md`
  authoring need — methodology propagation pipeline candidate for
  this WG's retro.
- **VX3.** "Pattern theater" detection (F4) was caught by
  V14 by reading the sketch carefully. The shape of detection
  ("Adapter applied to a non-adapter scenario") may generalize —
  candidate **verifier V14 refinement**: "Pattern claim with
  no clear interface-to-conform-to is pattern theater; reject
  or rewrite."

## References

- workgraph.md (this directory)
- plan.md (this directory)
- All seven task specs in `tasks/` (this directory)
- `vtf-methodologies/verifier/bugfix.md` V1-V15
- `vtf-methodologies/spec-author/bugfix.md` R1-R12
- `viloforge-platform/docs/engineering-principles.md` §3, §5.3
