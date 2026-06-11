# ADR-009 — Idempotency Key Implementation Contract

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-5 resolution, TDD API-5 |

## Context
`TDD.md` API-5 requires idempotency keys on resume upload (`POST /resumes/upload`) and match creation (`POST /matches`). Without a schema column backing this, the feature cannot be implemented. The `match_results` UNIQUE constraint provided functional deduplication but not client-driven idempotency.

## Decision

### Schema
Two tables receive idempotency key columns:

**`documents`**:
```sql
idempotency_key VARCHAR(255),
CONSTRAINT uq_documents_user_idempotency
    UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key)
```

**`match_results`** (in addition to existing `UNIQUE (resume_id, jd_id, model_version)`):
```sql
idempotency_key VARCHAR(255),
CONSTRAINT uq_matches_user_idempotency
    UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key)
```

`NULLS NOT DISTINCT` (Postgres 15+): multiple `NULL` keys are allowed per user; non-null keys must be unique per user.

### API Contract
- Header: `Idempotency-Key: <client-generated-string>` (optional; max 255 chars; any printable ASCII).
- Applicable endpoints: `POST /resumes/upload`, `POST /matches`.
- Behaviour when key matches an existing row for the same user within 24 hours:
  - Return HTTP 200 (not 202/201) with the **original response body** reconstructed from the existing rows.
  - Do not re-process. Do not enqueue a new ARQ task.
- Behaviour when key is absent or expired: standard processing, no idempotency guarantee.
- Key expiry: 24 hours from `documents.created_at` / `match_results.created_at`. Checked at the application layer.

### Implementation
1. BFF extracts the `Idempotency-Key` header and forwards it to FastAPI as `X-Idempotency-Key`.
2. FastAPI ingestion/matching service checks for an existing row with `(user_id, idempotency_key)` before processing.
3. If found and within 24 hours: return the reconstructed response immediately.
4. If not found: proceed with normal creation flow; store the key on the created row.

## Rationale
- Follows the widely understood Stripe idempotency key convention (clients already know this pattern).
- Column-level enforcement ensures the database rejects duplicates even if the application check races (UNIQUE constraint as the final safety net).
- Optional key preserves backward compatibility — clients that don't care about idempotency don't need to change.
- 24-hour TTL prevents the column from becoming a permanent deduplication log.

## Consequences
- Migrations for Phase 1 must include these columns even though the handlers aren't built until Phase 3 and Phase 6.
- Postgres 15+ required for `NULLS NOT DISTINCT`. This is met by the specified Postgres image in Compose.

## Revisit Trigger
If idempotency keys are needed on additional endpoints, extend the pattern to those tables following the same column + constraint approach.
