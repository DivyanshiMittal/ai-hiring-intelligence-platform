# ADR-013 — UUIDv7 via Application-Layer Generation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-3 resolution, TDD §0 |

## Context
`TDD.md §0` requires UUIDv7 identifiers for all primary keys ("time-sortable, non-enumerable"). PostgreSQL has no native v7 generator. The `pg_idkit` extension provides v7 but requires a custom Postgres Docker image. The Python `uuid` stdlib generates v4 only.

Three options were evaluated:
1. Application-layer generation via `python-uuid7` package.
2. `pg_idkit` extension on a custom Postgres image.
3. Downgrade to UUIDv4 (violates TDD §0).

## Decision
**Option 1: Application-layer generation via the `uuid7` Python package.**

```python
# app/core/uuidv7.py — the single ID generation point in the codebase
import uuid7 as _uuid7


def new_id() -> str:
    """Generate a time-sortable, non-enumerable UUIDv7 identifier."""
    return str(_uuid7.uuid7())
```

All SQLAlchemy ORM model base classes use:
```python
from app.core.uuidv7 import new_id
from sqlalchemy.orm import DeclarativeBase, mapped_column, Mapped
from sqlalchemy import UUID

class Base(DeclarativeBase):
    pass

class UUIDMixin:
    id: Mapped[str] = mapped_column(
        UUID(as_uuid=False),
        primary_key=True,
        default=new_id,
        # NO server_default — application always supplies the PK
    )
```

PostgreSQL column type is `UUID`. No `gen_random_uuid()` or `uuid_generate_v4()` server defaults anywhere. The application always supplies the PK before the INSERT.

The `uuid-ossp` extension is still installed (see ADR-015) but is not used for PK generation in application code. It is available for manual DBA queries and tooling.

## Rationale

**Why application-layer over `pg_idkit`:**
- No custom Postgres Docker image required.
- No operational risk from a compiled extension on the database server.
- Application controls ID generation, which is useful: the `resume_id` is known before the INSERT and can be passed to the `processing_jobs` row and the ARQ task in the same transaction (supporting ADR-017's write-order invariant).
- `python-uuid7` is a pure-Python package — auditable, no native compilation, works on all platforms.

**Why not UUIDv4:**
- Violates explicit TDD §0 requirement.
- Loses time-sortability, degrading `(user_id, created_at)` index efficiency and cursor pagination.
- Cannot be retrofitted without rewriting all PKs in all tables.

## Consequences
- `app/core/uuidv7.py` is the only permitted source of new IDs. Direct calls to `uuid.uuid4()` in model definitions are a lint error.
- Alembic migrations that define PK columns use `UUID` type with no `server_default`. The absence of `server_default` is intentional and must not be "fixed" by a well-meaning developer.
- Integration tests can verify time-sortability: IDs created in sequence must sort in creation order.

## Revisit Trigger
If Postgres ships a native `uuid_generate_v7()` function (plausible in Postgres 18+), migrate to server-side generation for consistency. The column type does not change; only the `default` vs `server_default` choice changes.
