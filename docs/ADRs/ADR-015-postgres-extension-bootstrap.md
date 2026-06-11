# ADR-015 — PostgreSQL Extension Bootstrap Strategy

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-2 resolution |

## Context
The schema requires `citext` (for `users.email`) and `uuid-ossp` (for tooling compatibility). These are PostgreSQL extensions that do not exist in a vanilla Postgres instance. Alembic autogenerate does not produce `CREATE EXTENSION` statements. Without explicit handling, `alembic upgrade head` fails on every fresh environment.

## Decision
Extension creation is guaranteed by **two independent mechanisms**:

### Mechanism 1: Docker Init Script
`infra/postgres/init.sql` runs once when the Docker volume is first created:
```sql
-- infra/postgres/init.sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;  -- observability
```
This covers local development and any environment where the database is bootstrapped from Docker.

### Mechanism 2: Alembic Baseline Migration
`alembic/versions/0001_baseline.py` `upgrade()` function:
```python
def upgrade() -> None:
    op.execute('CREATE EXTENSION IF NOT EXISTS citext')
    op.execute('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"')
    op.execute('CREATE EXTENSION IF NOT EXISTS pg_stat_statements')
    # Table definitions follow in subsequent migrations
```

`downgrade()` does **not** drop these extensions. Dropping is a manual DBA action because:
- Other objects (e.g., existing `citext` columns) depend on the extension.
- Dropping `pg_stat_statements` would lose query performance history.
- `IF NOT EXISTS` / `IF EXISTS` guards make re-runs idempotent.

### Why Both Mechanisms
- `init.sql` only runs on a fresh Docker volume. An existing staging or production database that already has the schema but hasn't run the new migration would not benefit from `init.sql`.
- Alembic runs on every database on `alembic upgrade head`. It is the deployment mechanism.
- Having both means: fresh environments get the extension from whichever mechanism runs first; existing environments get it from Alembic.

## Consequences
- The Alembic baseline migration must be the **first** migration in the chain (revision `0001`). No table creation migration may run before it.
- `pg_stat_statements` requires superuser to create. The Postgres Docker image runs as a superuser by default in development. Production must ensure the migration user has `CREATE EXTENSION` privilege or the DBA must create the extension manually before migrations run.
- The `uuid-ossp` extension is installed but not used for PK generation (see ADR-013). Its presence does not imply it is the PK generation mechanism.

## Revisit Trigger
If the platform migrates to a managed Postgres service (e.g., Cloud SQL, RDS, Supabase) where extensions are managed via the provider's UI or API rather than SQL, the Alembic migration statement may need to be removed or made conditional on the environment. Add a `SKIP_EXTENSION_CREATION=true` env flag at that point.
