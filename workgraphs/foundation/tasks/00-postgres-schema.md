---
id: t_K2pQ4r
slug: postgres-schema
title: vfobs Postgres schema bootstrap — vfobs schema, partitioned events table, DB user
workgraph: foundation
order: 0
required_tags:
  - executor
depends_on: []
target_repo: viloforge/vfobs
agent_model: claude-opus-4-7
judge: true
test_command:
  command: |
    git ls-remote --heads origin wg-vfobs-foundation/t0-postgres-schema 2>&1 | grep -q .
created_at: 2026-05-14T00:00:00Z
acceptance_criteria:
  - AC-T0-1 — `alembic` is configured in the vfobs repo with
    `alembic.ini` at repo root, `alembic/env.py`, and
    `alembic/versions/` directory. `env.py` reads the DB URL from
    `vfobs.config.Settings.database_url` (does NOT hardcode it).
  - AC-T0-2 — Migration `alembic/versions/0001_initial_schema.py`
    creates schema `vfobs`, creates the partitioned `vfobs.events`
    table per plan.md §D2, creates the parent table with
    `PARTITION BY RANGE (timestamp)`, and creates the first three
    monthly partitions (current month, next month, month after) to
    keep ingestion working for ~90 days without operator
    intervention. Indexes per plan.md §D2.
  - AC-T0-3 — Migration also creates a dedicated DB role
    `vfobs_app` (LOGIN, password externalised) with grants
    `USAGE ON SCHEMA vfobs`, `SELECT, INSERT ON ALL TABLES IN
    SCHEMA vfobs`, and `USAGE, SELECT ON ALL SEQUENCES IN SCHEMA
    vfobs`. No grants to `public`. Default privileges set so
    future tables/sequences in `vfobs` inherit the same grants.
  - AC-T0-4 — Migration is **idempotent at the role-creation step**
    via `DO $$ BEGIN IF NOT EXISTS … THEN CREATE ROLE … END IF; END $$;`.
    Schema + table + indexes use Alembic's normal create_* (Alembic
    tracks application via `alembic_version`). Re-running
    `alembic upgrade head` after first apply is a no-op (Alembic
    sees `head` already applied).
  - AC-T0-5 — Integration: `tests/integration/test_db_bootstrap.py`
    spins a disposable Postgres (via `pytest-postgresql` or
    docker-via-testcontainers), runs `alembic upgrade head`,
    asserts schema + table + role + indexes exist via direct
    information_schema queries. Re-runs `alembic upgrade head` and
    asserts no errors and no duplicate objects (idempotency).
  - AC-T0-6 — `vfobs_app` role has exactly the grants in AC-T0-3
    and no more (no `DELETE`, no `UPDATE`, no `DROP`). Integration
    test asserts this via `information_schema.role_table_grants`.
    Rationale: vfobs only ever appends; locking writes prevents
    accidental mutation of the event log.
  - AC-T0-7 — Branch `wg-vfobs-foundation/t0-postgres-schema` is
    pushed to `viloforge/vfobs` and a PR is open. (Externally-
    grounded per R1 — verified by the `test_command` gate above.)

---

# Spec

## Files touched

- `alembic.ini` (new) — Alembic config; `script_location = alembic`,
  `sqlalchemy.url` left blank (env.py resolves at runtime).
- `alembic/env.py` (new) — Standard Alembic env, imports
  `vfobs.config.get_settings()` to resolve `database_url`.
- `alembic/script.py.mako` (new) — Default Alembic template.
- `alembic/versions/0001_initial_schema.py` (new) — The schema +
  role migration.
- `src/vfobs/config.py` (new) — `Settings(BaseSettings)` with
  `database_url: PostgresDsn` and `get_settings()` LRU-cached
  factory. (Skeleton only; T3 adds more fields.)
- `tests/integration/test_db_bootstrap.py` (new) — bootstrap +
  idempotency + grants integration test.
- `pyproject.toml` (modify) — ensure
  `pydantic-settings>=2.0,<3.0` is in `[project] dependencies`
  (if not already from WG0); add `pytest-postgresql>=6.0,<7.0` to
  `[project.optional-dependencies] dev` for the integration test.

(NOT touched: `src/vfobs/events/`, `src/vfobs/repositories/`,
`src/vfobs/api/`, `charts/`, anything in `tests/unit/`, the SDK
subpackage. Everything in this task is data-layer bootstrap.)

## Implementation sketch

**Pattern: Adapter** (Alembic adapts our Python schema-definition
to whatever Postgres version the cluster runs). No application-layer
patterns; this task is pure DDL + bootstrap.

The migration file template:

```python
"""initial schema — vfobs.events partitioned table + vfobs_app role

Revision ID: 0001_initial_schema
Revises:
Create Date: 2026-05-14
"""
from alembic import op
import sqlalchemy as sa

revision = "0001_initial_schema"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE SCHEMA IF NOT EXISTS vfobs")

    op.execute("""
    DO $$ BEGIN
      IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'vfobs_app') THEN
        CREATE ROLE vfobs_app LOGIN PASSWORD :app_pw;
      END IF;
    END $$;
    """)
    # NOTE: in production the password comes from ESO → Vault.
    # The migration accepts a bound :app_pw parameter; CI/local
    # passes 'devpassword'. See plan.md §D6 + T5 for the prod path.

    op.execute("""
    CREATE TABLE vfobs.events (
        id BIGSERIAL NOT NULL,
        v SMALLINT NOT NULL DEFAULT 1,
        workgraph_id TEXT NOT NULL,
        task_id TEXT NULL,
        agent_id TEXT NULL,
        trace_id TEXT NULL,
        source TEXT NOT NULL,
        type TEXT NOT NULL,
        timestamp TIMESTAMPTZ NOT NULL,
        data JSONB NOT NULL DEFAULT '{}'::jsonb,
        classification TEXT NOT NULL DEFAULT 'internal',
        org_id TEXT NOT NULL DEFAULT 'viloforge',
        cluster_id TEXT NOT NULL DEFAULT 'vafi-dev',
        created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
        CONSTRAINT events_type_namespace_chk CHECK (
          type ~ '^(workgraph|task|harness|gate|judge|anomaly)\\.'
        ),
        CONSTRAINT events_classification_chk CHECK (
          classification IN ('public','internal','secret')
        ),
        PRIMARY KEY (id, timestamp)
    ) PARTITION BY RANGE (timestamp);
    """)

    # First three monthly partitions
    for i in range(3):
        op.execute(_partition_for_month_offset(i))

    op.execute("CREATE INDEX events_workgraph_id_id_idx ON vfobs.events (workgraph_id, id)")
    op.execute("CREATE INDEX events_task_id_id_idx ON vfobs.events (task_id, id) WHERE task_id IS NOT NULL")
    op.execute("CREATE INDEX events_type_id_idx ON vfobs.events (type, id)")
    op.execute("CREATE INDEX events_timestamp_idx ON vfobs.events (timestamp)")
    op.execute("CREATE INDEX events_org_cluster_idx ON vfobs.events (org_id, cluster_id, id)")

    op.execute("GRANT USAGE ON SCHEMA vfobs TO vfobs_app")
    op.execute("GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA vfobs TO vfobs_app")
    op.execute("GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA vfobs TO vfobs_app")
    op.execute("ALTER DEFAULT PRIVILEGES IN SCHEMA vfobs GRANT SELECT, INSERT ON TABLES TO vfobs_app")
    op.execute("ALTER DEFAULT PRIVILEGES IN SCHEMA vfobs GRANT USAGE, SELECT ON SEQUENCES TO vfobs_app")


def downgrade() -> None:
    op.execute("DROP SCHEMA IF EXISTS vfobs CASCADE")
    op.execute("DROP ROLE IF EXISTS vfobs_app")
```

The helper `_partition_for_month_offset(i)` computes the bounds for
`events_<YYYY_MM>` (e.g., `events_2026_05`). Implementation uses
`datetime.date.today()` + `dateutil.relativedelta` for the boundaries.

The integration test in `tests/integration/test_db_bootstrap.py`:

```python
@pytest.mark.integration
def test_bootstrap_creates_schema_table_role(postgresql):
    settings = Settings(database_url=postgresql_dsn(postgresql))
    run_alembic_upgrade_head(settings)

    assert _schema_exists(postgresql, "vfobs")
    assert _table_exists(postgresql, "vfobs", "events")
    assert _role_exists(postgresql, "vfobs_app")
    assert _index_exists(postgresql, "vfobs", "events_workgraph_id_id_idx")
    # ... etc.


@pytest.mark.integration
def test_bootstrap_is_idempotent(postgresql):
    settings = Settings(database_url=postgresql_dsn(postgresql))
    run_alembic_upgrade_head(settings)
    run_alembic_upgrade_head(settings)   # second run must not raise
    # assert exactly one alembic_version row
    rows = _fetch(postgresql, "SELECT count(*) FROM alembic_version")
    assert rows[0][0] == 1


@pytest.mark.integration
def test_vfobs_app_grants_are_minimal(postgresql):
    settings = Settings(database_url=postgresql_dsn(postgresql))
    run_alembic_upgrade_head(settings)
    grants = _fetch(postgresql, """
        SELECT privilege_type
        FROM information_schema.role_table_grants
        WHERE grantee = 'vfobs_app' AND table_schema = 'vfobs'
    """)
    privs = {row[0] for row in grants}
    assert privs == {"SELECT", "INSERT"}  # no DELETE, UPDATE, etc.
```

## Fail-loud directive (R3)

If any required step (alembic apply, branch push, PR creation)
cannot be completed due to missing credentials, missing tools, or
blocked external dependencies: report the failure explicitly in
your completion notes — do not rationalize partial completion as
success. The judge will fail tasks that report dishonest success;
tasks that report failure honestly can be reworked.

# Constraints

- Migration must work against Postgres 14+ (vtaskforge's shared
  instance per OIQ2). No PG15+ features (e.g., `MERGE` syntax,
  `SELECT … RETURNING …` is fine).
- `vfobs_app` role's grants are **minimum-necessary**: SELECT +
  INSERT only on the `vfobs.events` table family + the BIGSERIAL
  sequence. No DELETE, UPDATE, DROP, TRUNCATE.
- Monthly partition creation must use a stable naming convention
  (`events_YYYY_MM`) so a future partition-management worker can
  enumerate them deterministically.
- The migration must not require operator-supplied parameters at
  runtime — `:app_pw` is filled by Alembic config in `env.py` from
  an env var (`VFOBS_APP_DB_PASSWORD`), not interactive prompt.

# Out of scope

- Automatic monthly partition creation worker. T0 creates the first
  three; a follow-up `kind: infrastructure` workgraph adds the
  scheduled creator. (Manual operator intervention bridges if
  needed before then.)
- Retention/archival of old partitions. Same follow-up.
- The actual deployment of this migration to vtaskforge's shared
  Postgres. Deployment is T5 + T6 (chart + scenario test);
  operator-or-ops will run the migration against the real
  shared Postgres separately as a one-time op.
- Read-only `vfobs_read` role for the WG2 read API. WG2 adds it.

# References

- workgraph.md, plan.md (this directory)
- `viloforge-platform/docs/pipeline-observability-DESIGN.md` §C, §D
- Postgres docs: declarative partitioning
  https://www.postgresql.org/docs/14/ddl-partitioning.html
- Alembic docs: tutorial
  https://alembic.sqlalchemy.org/en/latest/tutorial.html
- vtaskforge's existing migration setup at
  `vtaskforge/src/vtaskforge/settings/` (reference for env-resolved DB URL)
