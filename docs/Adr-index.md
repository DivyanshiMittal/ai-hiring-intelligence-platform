# ADR Index — Hiring-Copilot
 
| | |
|---|---|
| **Project** | Hiring-Copilot |
| **Document** | Architecture Decision Record Index |
| **Version** | 1.0 |
| **Date** | 2026-06-10 |
| **Status** | Complete — 22 ADRs |
 
> All ADRs are in `docs/ADRs/`. This index is the single entry point. ADRs are append-only; a superseded ADR is marked here but its file is never deleted.
 
---
 
## Core Architecture (Source: architecture.md AD-1 through AD-6)
 
| ADR | Title | Status | Phase Impact |
|---|---|---|---|
| [ADR-001](ADR-001-modular-monolith.md) | Modular Monolith over Microservices | Accepted | All phases |
| [ADR-002](ADR-002-nextjs-bff.md) | Next.js BFF Fronts All Client Traffic; FastAPI Private | Accepted | All phases |
| [ADR-003](ADR-003-async-first.md) | Async-First API Design for Slow AI Operations | Accepted | P3 onward |
| [ADR-004](ADR-004-langgraph-policy.md) | LangGraph for Stateful/Branching Flows Only | Accepted | P4, P6, P9, P10 |
| [ADR-005](ADR-005-sse-over-websockets.md) | SSE over WebSockets for Evaluation Streaming | Accepted | P3, P9 |
| [ADR-006](ADR-006-llm-gateway.md) | All LLM Calls via Internal Gateway | Accepted | P4 onward |
 
## Critical Issue Resolutions (Source: critical-issues-resolution.md C-1 through C-7)
 
| ADR | Title | Status | Resolves | Phase Impact |
|---|---|---|---|---|
| [ADR-007](ADR-007-worker-image-cmd-override.md) | Worker Uses API Image with CMD Override | Accepted | C-1 | P1 |
| [ADR-009](ADR-009-idempotency-keys.md) | Idempotency Key Implementation Contract | Accepted | C-5 | P1 (schema), P3, P6 |
| [ADR-012](ADR-012-bff-sse-proxy.md) | BFF SSE Proxy Implementation Contract | Accepted | C-4 | P1 (scaffold), P3, P9 |
| [ADR-013](ADR-013-uuidv7-application-layer.md) | UUIDv7 via Application-Layer Generation | Accepted | C-3 | P1 (all models) |
| [ADR-015](ADR-015-postgres-extension-bootstrap.md) | PostgreSQL Extension Bootstrap Strategy | Accepted | C-2 | P1 |
| [ADR-016](ADR-016-qdrant-tenant-isolation.md) | Qdrant Tenant Isolation via TenantScopedQdrantRepository | Accepted | C-6 | P5 onward |
| [ADR-017](ADR-017-processing-jobs-write-order.md) | Processing Jobs Write-Order Invariant | Accepted | C-7 | P1 (schema), P3 onward |
 
## Recommended Issue Resolutions (Source: pre-implementation review R-1 through R-10)
 
| ADR | Title | Status | Resolves | Phase Impact |
|---|---|---|---|---|
| [ADR-008](ADR-008-bff-proxy-design.md) | BFF Proxy: Generic Reverse Proxy with Envelope Normalisation | Accepted | R-1 | P1 |
| [ADR-010](ADR-010-taxonomy-seed.md) | Skills Taxonomy: MVT Stub P1, Full ESCO P4 | Accepted | R-3 | P1, P4 |
| [ADR-011](ADR-011-followup-write-model.md) | Follow-up Questions Inserted On-the-Fly | Accepted | R-10 | P9 |
| [ADR-014](ADR-014-clamav-sidecar.md) | ClamAV Sidecar: Scaffolded P1, Activated P5 | Accepted | SEC-5 | P1, P5 |
| [ADR-018](ADR-018-model-version-format.md) | model_version Format: provider:model_id:prompt-vN | Accepted | R-2 | P4 onward |
| [ADR-019](ADR-019-arq-idempotency-pattern.md) | ARQ Task Handler Idempotency Gate-Then-Work Pattern | Accepted | R-4 | P3 onward |
| [ADR-020](ADR-020-otel-trace-propagation.md) | OTel Trace Propagation into ARQ Workers | Accepted | R-6 | P1 (scaffold), P3 onward |
| [ADR-021](ADR-021-llm-audit-log-pii.md) | LLM Audit Log PII Redaction Contract | Accepted | R-7 | P4 onward |
| [ADR-022](ADR-022-structured-logging.md) | Structured JSON Logging via structlog and pino | Accepted | O-6 | P1 |
 
---
 
## Decision Coverage Map
 
This table maps each architectural concern to the ADR(s) that govern it.
 
| Concern | ADR(s) |
|---|---|
| Deployment topology | ADR-001, ADR-002 |
| Container / image strategy | ADR-007, ADR-014 |
| Database identity / PKs | ADR-013 |
| Database extensions | ADR-015 |
| Database schema — idempotency | ADR-009 |
| Database schema — model versioning | ADR-018 |
| Async job lifecycle | ADR-003, ADR-017, ADR-019 |
| AI orchestration (LangGraph) | ADR-004 |
| LLM interaction | ADR-006, ADR-021 |
| Streaming (SSE) | ADR-005, ADR-012 |
| BFF proxy | ADR-002, ADR-008 |
| Qdrant — tenant isolation | ADR-016 |
| Qdrant — taxonomy / seed | ADR-010 |
| Interview adaptivity | ADR-011 |
| Observability — tracing | ADR-020 |
| Observability — logging | ADR-022 |
| Security — file upload | ADR-014 |
 
---
 
## Open Decisions (Not Yet Formalised as ADRs)
 
These items were flagged as Optional in the pre-implementation review. They do not block implementation but should be resolved before the phase that requires them.
 
| Item | Phase Required By | Notes |
|---|---|---|
| Readiness index formula | Phase 10 | Suggested: 0.4×match + 0.4×interview + 0.2×roadmap completion |
| Concurrent SSE connection rate limit | Phase 9 | Redis counter; max 3 active eval streams per user |
| RSC direct FastAPI vs BFF proxy | Phase 1 | ADR-002 notes: RSC uses server-side api-client with service JWT; no extra HTTP hop |
| Resume re-upload version strategy | Phase 3 | New row per upload; version increments per user+document lineage |
| LangGraph checkpointer connection budget | Phase 9 | Dedicated pool; budget arithmetic before P9 starts |
 
*Formalise these as ADRs at the start of the relevant phase, not retroactively.*
 