# Critical Issues Resolution Record

| | |
|---|---|
| **Project** | Hiring-Copilot — AI Hiring Intelligence Platform |
| **Document** | Critical Issues Resolution Record |
| **Version** | 1.0 |
| **Status** | All resolutions accepted — implementation not yet begun |
| **Date** | 2026-06-10 |
| **Author** | Principal AI Architect / Technical Lead |
| **Audience** | All implementing engineers; staff reviewers |

> **Purpose.** Seven critical issues were identified during pre-implementation analysis. This document records each issue, its accepted resolution, the architectural assumptions it updates, the ADR it produces, and the phases it affects. No implementation may begin until all resolutions are reflected in the ADR set, bootstrap analysis, and project journal.

---

## C-1 — Worker Image / Entrypoint Undefined

### Issue Statement
The architecture listed three Docker images (`web`, `api`, `worker`) as Phase 1 deliverables (DEP-1). Worker code lives inside `apps/api/app/workers/` with no `Dockerfile.worker` and no specified entrypoint strategy. A separate worker image causes dependency drift; the same-image approach needed an explicit CMD override strategy.

### Risk if Unresolved
Separate images drift in Python dependencies, causing silent incompatibilities between the API and the worker processing the same domain objects. CI would have no defined build target. Phase 1's "docker compose up" success criterion would be undefined.

### Accepted Resolution
The `worker` service uses **the same built image as `api`** with the `command` field in `docker-compose.yml` overriding the default `uvicorn` entrypoint:

```yaml
worker:
  image: hiring-copilot-api        # same image, built once
  command: arq app.workers.main.WorkerSettings
```

The `Dockerfile.api` default CMD remains `uvicorn`. There is no `Dockerfile.worker`. CI builds one image; Compose uses it for both services. DEP-1 is updated to read **two application images** (`web`, `api`), with the `api` image reused for `worker`.

### Assumptions Updated
- `TDD.md` DEP-1: "Three images" → "Two application images; `api` image reused for `worker` via CMD override."
- `architecture.md` §4 `infra/docker/`: `Dockerfile.worker` entry removed.

### ADR Produced
ADR-007 (updated) — Worker Uses API Image with CMD Override

### Phase Impact
Phase 1 (critical: Compose definition, CI build). All subsequent phases inherit this pattern.

---

## C-2 — PostgreSQL `citext` Extension Missing from Migration Strategy

### Issue Statement
The schema specifies `users.email citext UNIQUE`. The `citext` type requires `CREATE EXTENSION IF NOT EXISTS citext`. Alembic autogenerate does not produce extension statements. Without explicit handling, `alembic upgrade head` fails on every fresh environment with `type "citext" does not exist`.

### Risk if Unresolved
Complete migration chain failure on fresh local, CI, staging, and production databases. A developer with `citext` already present on their personal Postgres would not catch the failure until CI or a colleague's environment.

### Accepted Resolution
Extension creation is written in **two places**:

1. **`infra/postgres/init.sql`** — Docker init script, runs once on volume creation:
   ```sql
   CREATE EXTENSION IF NOT EXISTS citext;
   CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
   ```

2. **`alembic/versions/0001_baseline.py` `upgrade()`**:
   ```python
   op.execute('CREATE EXTENSION IF NOT EXISTS citext')
   op.execute('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"')
   ```
   `downgrade()` does **not** drop extensions — dropping is a manual DBA action because other objects may depend on them.

### Assumptions Updated
- `architecture.md` §3.3 Principles: "`citext` and `uuid-ossp` extensions are guaranteed by the baseline Alembic migration."
- `TDD.md` DEP-5: "Baseline migration MUST create `citext` and `uuid-ossp` before any table definition."

### ADR Produced
ADR-015 (new) — PostgreSQL Extension Bootstrap Strategy

### Phase Impact
Phase 1 (critical: baseline migration, init.sql). Phase 2 (users table depends on citext).

---

## C-3 — UUIDv7 Has No Implementation Path

### Issue Statement
`TDD.md §0` requires UUIDv7 identifiers ("time-sortable, non-enumerable"). PostgreSQL has no native `uuid_generate_v7()`. The `pg_idkit` extension provides v7 but requires a custom Postgres image. The Python `uuid` module generates v4 only. No library or generation strategy was specified anywhere in the documents.

### Risk if Unresolved
Silent default to UUIDv4 violates an explicit TDD requirement. Loss of time-sortability degrades cursor pagination and `(user_id, created_at)` index efficiency. Retrofitting all PKs post-implementation requires a migration rewriting every row in every table.

### Accepted Resolution
**Application-layer generation via the `uuid7` Python package.**

```python
# app/core/uuidv7.py  — the single generation point
import uuid7 as _uuid7

def new_id() -> str:
    return str(_uuid7.uuid7())
```

- PostgreSQL column type remains `UUID`.
- All SQLAlchemy model PKs use `default=new_id` (not `server_default`).
- Application always supplies the PK before any INSERT.
- `uuid-ossp` is still installed (useful for manual DB tooling and ad-hoc queries) but is **not** used for PK generation in application code.
- No custom Postgres image required.

### Assumptions Updated
- `TDD.md §0`: "UUIDv7 generated application-side via `uuid7` Python package; column type is `UUID`; Postgres does not generate PKs."
- `architecture.md §3.3`: "All PKs supplied by application layer using `uuid7.uuid7()`. No `gen_random_uuid()` or `uuid_generate_v4()` server defaults."

### ADR Produced
ADR-013 (new) — UUIDv7 via Application-Layer Generation

### Phase Impact
Phase 1 (critical: base model class, all migrations). Every subsequent phase inherits the base model.

---

## C-4 — SSE Proxy Through Next.js BFF Unspecified

### Issue Statement
The architecture required all browser traffic to route through the Next.js BFF (AD-2) and evaluation/job-progress to stream via SSE (AD-5). Next.js Route Handlers have runtime-specific SSE proxy requirements: the Edge runtime cannot proxy SSE; specific HTTP headers must prevent proxy/CDN buffering; without a heartbeat, long-lived connections time out silently at 30 seconds.

### Risk if Unresolved
- Developer uses Edge runtime → hard failure, no clear error.
- Proxy/CDN buffers stream → client receives full response at once, defeating SSE.
- No heartbeat → 30-second timeouts break sessions mid-evaluation.
- Developer opens FastAPI directly from browser to work around it → violates AD-2, exposes service tokens.

### Accepted Resolution
All BFF SSE proxy routes MUST conform to the following contract:

1. **Runtime declaration** — mandatory at top of every SSE route file:
   ```typescript
   export const runtime = 'nodejs';
   ```

2. **Required response headers** on every proxied SSE response:
   ```
   Content-Type: text/event-stream
   Cache-Control: no-cache, no-transform
   X-Accel-Buffering: no
   Connection: keep-alive
   ```

3. **Heartbeat** — BFF injects a comment event (`:\n\n`) every 15 seconds into the stream. FastAPI does not need to produce heartbeats; the BFF owns this responsibility.

4. **Implementation pattern** — BFF uses `fetch()` with the service JWT to call the private FastAPI SSE endpoint, pipes the response body through a `TransformStream` that injects heartbeats, and returns `new Response(transformedStream, { headers })`.

5. **FastAPI SSE endpoints** remain private-network only. The browser never calls them directly under any circumstances.

### Assumptions Updated
- `architecture.md §1` BFF layer: add SSE runtime constraint, header requirements, heartbeat contract.
- `TDD.md` AD-5: add the runtime, header, and heartbeat requirements.

### ADR Produced
ADR-012 (new) — BFF SSE Proxy Implementation Contract

### Phase Impact
Phase 1 (BFF scaffold establishes Node.js runtime pattern). Phase 3 (critical: first SSE usage). Phase 9 (evaluation streaming).

---

## C-5 — Idempotency Key Column Absent from Schema

### Issue Statement
API-5 and the cross-cutting functional requirements explicitly required idempotency keys on resume upload and match creation. The `match_results` UNIQUE constraint provided functional deduplication but not client-driven RFC-style idempotency. The `documents` table had no idempotency mechanism. No column was specified in the schema, making the migration impossible to write.

### Risk if Unresolved
A client retrying a failed upload creates duplicate `documents` and `resumes` rows, triggering duplicate parse jobs and double-charging token budgets. The API contract promises idempotency keys but has no backing store.

### Accepted Resolution
**Schema additions:**

`documents` table:
```sql
idempotency_key VARCHAR(255),
CONSTRAINT uq_documents_user_idempotency
    UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key)
```

`match_results` table (in addition to the existing functional unique constraint):
```sql
idempotency_key VARCHAR(255),
CONSTRAINT uq_matches_user_idempotency
    UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key)
```

`NULLS NOT DISTINCT` (Postgres 15+) allows multiple `NULL` keys for the same user while enforcing uniqueness for non-null keys.

**API contract:**
- `POST /resumes/upload` and `POST /matches` accept an optional `Idempotency-Key: <string>` request header (Stripe convention).
- If a matching key exists for the same user within 24 hours, the API returns the original response (HTTP 200) without re-processing.
- If no key is supplied, idempotency guarantees are not provided for that request.
- Key expiry: 24 hours from `created_at` of the original row, enforced at the application layer.

### Assumptions Updated
- `architecture.md §3.2`: `idempotency_key` column added to `documents` and `match_results` schema blocks.
- `architecture.md §5.1`: `Idempotency-Key` header documented in upload and match contracts.
- `TDD.md` API-5: column-level specification and 24-hour expiry added.

### ADR Produced
ADR-009 (new) — Idempotency Key Implementation Contract

### Phase Impact
Phase 1 (critical: schema/migration must include columns). Phase 3 (upload handler). Phase 6 (match creation handler).

---

## C-6 — Qdrant Tenant Isolation Has No Enforcement Mechanism

### Issue Statement
The architecture required server-injected `user_id` filters on all Qdrant queries (SEC-2, Architecture §7) but specified no enforcement mechanism. Without structural enforcement this is a convention, not a guarantee. A single omission creates a cross-tenant data leak in a vector search context — silently returning another user's private resume content as "similar" results.

### Risk if Unresolved
Cross-tenant data leakage invisible in single-user testing. Security audit or production incident. Vector search leakage is particularly severe because the leaked data appears as semantically relevant results, not as an obvious error.

### Accepted Resolution
All Qdrant access is centralized behind a `TenantScopedQdrantRepository` class with the following structural guarantees:

1. Constructor accepts `actor: ActorContext`. The `user_id` is extracted once at construction; it is never a parameter on query methods.
2. The `_tenant_filter()` method is private and composed into every query. It is structurally impossible for a caller to omit it.
3. No code outside `app/retrieval/qdrant_repository.py` may import the Qdrant client. This is enforced by a linting import boundary rule in `pyproject.toml`.
4. A mandatory integration test `test_cross_tenant_isolation` exists: creates data for user A, queries as user B, asserts empty results.

```python
# Structural contract (illustrative — not implementation code)
class TenantScopedQdrantRepository:
    def __init__(self, client: QdrantClient, actor: ActorContext) -> None:
        self._client = client
        self._user_id = actor.user_id  # extracted once; never re-exposed

    def _tenant_filter(self) -> Filter:
        return Filter(must=[FieldCondition(
            key="user_id", match=MatchValue(value=self._user_id)
        )])

    async def search_resume_chunks(
        self, query_vector: list[float], top_k: int
    ) -> list[ScoredPoint]:
        return await self._client.search(
            collection_name=CollectionAlias.RESUME_CHUNKS,
            query_vector=query_vector,
            query_filter=self._tenant_filter(),  # always injected; caller cannot skip
            limit=top_k,
        )
```

### Assumptions Updated
- `architecture.md §7`: "All Qdrant access MUST go through `TenantScopedQdrantRepository`. Direct Qdrant client import outside `app/retrieval/` is a lint error."
- `TDD.md` SEC-2: structural enforcement via `TenantScopedQdrantRepository` and import boundary linting added.

### ADR Produced
ADR-016 (new) — Qdrant Tenant Isolation via TenantScopedQdrantRepository

### Phase Impact
Phase 5 (critical: first Qdrant implementation). Phases 6, 7, 8, 9 (all retrieval operations).

---

## C-7 — `processing_jobs` Write Ordering Undefined (Race Condition)

### Issue Statement
The write ordering between `documents`, `resumes`, `processing_jobs`, and the ARQ enqueue call was never specified. If ARQ enqueues before DB rows commit, the worker attempts to read rows that don't yet exist. If `processing_jobs` is written after ARQ enqueues, the worker can update a job that hasn't been created yet.

### Risk if Unresolved
Intermittent foreign key violations in the worker — appearing only under load or on slow Postgres connections, very difficult to reproduce and debug. A worker completing before its `processing_jobs` row commits leaves the job permanently stuck in `QUEUED`.

### Accepted Resolution
**Mandatory write order** for all ingestion operations:

```
BEGIN TRANSACTION
  Step 1: INSERT INTO documents (...)           → produces document_id
  Step 2: INSERT INTO resumes (status=PENDING)  → produces resume_id
  Step 3: INSERT INTO processing_jobs (
              status=QUEUED,
              entity_id=resume_id,
              job_type='parse_resume'
          )
COMMIT TRANSACTION

Step 4: enqueue ARQ task                        ← OUTSIDE and AFTER commit
```

**The ARQ enqueue call is always outside the transaction and always after commit.** This is an inviolable invariant.

Consequences:
- Transaction failure → no task enqueued → no worker reads non-existent rows.
- ARQ enqueue failure after commit → `processing_jobs` row exists in `QUEUED` → manually re-enqueueable; no data loss.
- Worker's first action on pickup: `SELECT ... FOR UPDATE` on `processing_jobs` to atomically transition `QUEUED → RUNNING`.

This same ordering pattern is mandatory for **all** async entity creation flows:
- JD parsing: `job_descriptions` → `processing_jobs` → commit → enqueue
- Match scoring: `match_results(PENDING)` → `processing_jobs` → commit → enqueue
- Roadmap generation: `learning_roadmaps(PENDING)` → `processing_jobs` → commit → enqueue
- User deletion: `processing_jobs(type=delete_user)` → commit → enqueue

### Assumptions Updated
- `architecture.md §2.2` flow diagram: commit precedes enqueue (explicit annotation added).
- `TDD.md` DD-5: "ARQ enqueue MUST occur after the transaction containing the `processing_jobs` row has committed."
- `TDD.md` NFR-3: reference to mandatory write-order invariant added.

### ADR Produced
ADR-017 (new) — Processing Jobs Write-Order Invariant

### Phase Impact
Phase 1 (critical: `processing_jobs` schema established). Phase 3 (critical: first use). Phases 4, 6, 9 (every subsequent async operation).

---

## Consolidated Assumption Changes

| Location | Before | After |
|---|---|---|
| `TDD.md` DEP-1 | "Three images: web, api, worker" | "Two images: web, api. API image reused for worker via CMD override." |
| `architecture.md §4` infra/docker/ | Dockerfile.worker listed | Dockerfile.worker removed |
| `TDD.md` DEP-5 | No extension creation | Baseline migration MUST create citext and uuid-ossp |
| `architecture.md §3.3` | Silent on extensions | citext and uuid-ossp guaranteed by baseline migration |
| `TDD.md §0` | "UUIDv7" with no implementation path | "UUIDv7 via uuid7 Python package; application-layer generation; no server defaults" |
| `architecture.md §3.3` | Silent on PK generation | "All PKs from application-layer uuid7.uuid7(). No gen_random_uuid() server defaults." |
| `architecture.md §1` BFF | SSE mentioned but not constrained | Node.js runtime required; headers specified; 15s heartbeat required |
| `TDD.md` AD-5 | SSE chosen | Add runtime, header, and heartbeat requirements |
| `architecture.md §3.2` documents | No idempotency_key column | idempotency_key VARCHAR(255), UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key) |
| `architecture.md §3.2` match_results | Functional unique constraint only | Add idempotency_key column with same constraint pattern |
| `architecture.md §5.1` | No Idempotency-Key header | POST /resumes/upload and POST /matches accept Idempotency-Key header |
| `architecture.md §7` | "server-injected filter" as convention | All access via TenantScopedQdrantRepository; direct import is lint error |
| `TDD.md` SEC-2 | Convention-based filter | Structural enforcement + import boundary linting + mandatory integration test |
| `architecture.md §2.2` | Enqueue shown without ordering | Commit precedes enqueue; annotated on flow diagram |
| `TDD.md` DD-5 | Async via processing_jobs | ARQ enqueue MUST occur after containing transaction commits |

---

## Resolution Sign-Off

| Issue | Resolution | Status |
|---|---|---|
| C-1 Worker image | API image + CMD override | Accepted |
| C-2 citext extension | init.sql + baseline migration | Accepted |
| C-3 UUIDv7 | uuid7 Python package, app-layer | Accepted |
| C-4 SSE proxy | Node.js runtime + headers + heartbeat | Accepted |
| C-5 Idempotency key | Column + NULLS NOT DISTINCT constraint | Accepted |
| C-6 Qdrant isolation | TenantScopedQdrantRepository | Accepted |
| C-7 Write order | entity → job → commit → enqueue | Accepted |

*Document frozen after sign-off. Subsequent changes require a new ADR and journal entry.*
