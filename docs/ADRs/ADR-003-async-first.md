# ADR-003 — Async-First API Design for Slow AI Operations

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-3, TDD DD-5, NFR-1 |

## Context
Several platform operations involve multi-second LLM calls: resume parsing (~5–15 s), match scoring (~4–6 s), roadmap generation (~10–20 s). The question is whether to block the HTTP request until these complete or return immediately and drive progress asynchronously.

## Decision
All slow AI operations follow an **async-first pattern**:
1. The HTTP request validates input, writes entity rows and a `processing_jobs` row within one transaction, commits, then enqueues an ARQ task (see ADR-017 for the mandatory write-order invariant).
2. The response returns `202 Accepted` with `{ entity_id, job_id }` immediately — always within 500 ms.
3. The ARQ worker performs the LLM work and updates the entity + job rows on completion.
4. The client tracks progress via `GET /jobs/{id}` (poll) or `GET /jobs/{id}/stream` (SSE).

Synchronous endpoints are reserved for fast reads (< 300 ms p95) and cheap mutations that do not call the LLM.

## Rationale
- Keeps all HTTP endpoints within the 300 ms p95 non-LLM SLO (NFR-1).
- Avoids thread exhaustion from long-held HTTP connections.
- Makes retries, idempotency, and failure recovery explicit via the `processing_jobs` table.
- Provides real progress feedback to the UI (not a spinner with no information).

## Rejected Alternative
Synchronous LLM calls in request handlers: timeouts, thread exhaustion under concurrent load, no retry mechanism, poor UX.

## Consequences
- Clients must handle eventual consistency: a `POST /resumes/upload` response does not contain the parsed resume; the client subscribes for progress.
- The UI uses optimistic rendering where appropriate (show uploaded filename immediately; show parsed skills when the job completes).

## Revisit Trigger
If a new operation is demonstrably fast enough (< 300 ms p95 including LLM call) it may be implemented synchronously. This is unlikely for any Gemini 2.5 Pro call.
