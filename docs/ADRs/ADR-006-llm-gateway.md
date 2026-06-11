# ADR-006 — All LLM Calls via Internal Gateway

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-6, TDD NFR-7 |

## Context
The platform makes LLM calls from multiple domains: ingestion (parsing), matching (scoring), interview (generation, evaluation), roadmap (RAG). The question is whether each domain calls Gemini directly or through a shared abstraction.

## Decision
All model calls route through `app/llm/gateway.py`. Domain code imports from `app/llm/`; it does not import the Gemini SDK directly. The gateway enforces:

1. **Structured output** — every call specifies a Pydantic response schema; the gateway uses Gemini function-calling / JSON schema mode and validates the response before returning it to the caller.
2. **Token accounting** — every call is logged to `llm_audit_log` with `(feature, model, prompt_tokens, completion_tokens, cost_usd, latency_ms, cache_hit, model_version)`. Prompt text is never logged (PII risk — see ADR-R7 in critical-issues-resolution.md).
3. **Response caching** — responses cached in Redis keyed by `SHA256(prompt_content) + model_version`. Cache hits do not call Gemini and cost $0.
4. **Retry / backoff / circuit breaker** — exponential backoff on transient errors; circuit breaker trips after consecutive failures and degrades gracefully (jobs are re-queued, not dropped).
5. **Per-feature concurrency caps** — Redis semaphores limit concurrent Gemini calls per feature to prevent cost runaway.
6. **Model swap** — changing the model requires updating a single config value; domain code is provider-agnostic.

## Rationale
- One place for cost control, observability, and provider abstraction.
- Domains remain testable without LLM — the gateway is mockable at the interface boundary.
- PII redaction enforced at the gateway; no domain needs to remember to redact.

## Rejected Alternative
Direct SDK calls in domain code: provider lock-in, scattered cost controls, untestable business logic, inconsistent audit logging.

## Consequences
Domain code never sees raw Gemini responses — only validated Pydantic objects. If the Gemini SDK changes its API, only the gateway is updated.

## Revisit Trigger
Never. The abstraction is cheap and the seam is intentional.
