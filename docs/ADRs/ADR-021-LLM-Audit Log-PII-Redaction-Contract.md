# ADR-021 — LLM Audit Log PII Redaction Contract
 
| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-7, TDD SEC-3, NFR-6 |
 
## Context
SEC-3 requires PII to be redacted before any LLM audit log or trace. Without a concrete mechanism, developers may log prompt content (which contains full resume text including names, emails, phone numbers) or complete the work "when we have time."
 
## Decision
 
### What the `llm_audit_log` Table Stores
The table stores **only** telemetry data — never prompt or completion content:
 
```
user_id, feature, model, prompt_tokens, completion_tokens,
cost_usd, latency_ms, cache_hit, model_version, created_at
```
 
Prompt text, completion text, and all input/output content are **never persisted to any table.** This is a hard rule enforced by the gateway's logging implementation.
 
### What OTel Traces Store
OTel spans for LLM calls carry:
- `llm.model` (string)
- `llm.feature` (string)
- `llm.prompt_tokens` (int)
- `llm.completion_tokens` (int)
- `llm.latency_ms` (int)
- `llm.cache_hit` (bool)
Span attributes do **not** include prompt text or completion text.
 
### Prompt Debugging (Development Only)
A `LOG_PROMPTS` environment variable (default: `false`, hardcoded `false` in production config):
```python
# app/core/config.py
class LLMSettings(BaseSettings):
    log_prompts: bool = False  # NEVER true in production
 
    @validator("log_prompts")
    def enforce_false_in_production(cls, v, values):
        if values.get("environment") == "production" and v:
            raise ValueError("log_prompts must be false in production")
        return v
```
 
When `LOG_PROMPTS=true` (development only), prompt content is written to **stderr** (not to the database, not to the OTel exporter). It is visible in `docker compose logs api` but is not persisted anywhere.
 
### Gateway Enforcement
The gateway's `_log_call()` method is the only place where `llm_audit_log` rows are written. It receives only the telemetry dict, never the prompt. The prompt is used for the LLM call and then goes out of scope:
 
```python
async def call(self, prompt: str, schema: type[T], feature: str) -> T:
    start = time.monotonic()
    response = await self._client.generate(prompt, response_schema=schema)
    latency_ms = int((time.monotonic() - start) * 1000)
 
    # Prompt goes out of scope here. Only telemetry is logged.
    await self._log_call(
        feature=feature,
        prompt_tokens=response.usage.prompt_tokens,
        completion_tokens=response.usage.completion_tokens,
        latency_ms=latency_ms,
        cache_hit=False,
    )
    return self._validate(response, schema)
```
 
## Consequences
- LLM call debugging in production must rely on token counts and latency, not prompt content. This is the correct production posture.
- If a specific prompt needs investigation in production (e.g., a parsing failure), the input can be reconstructed from the entity's `parsed_json` or `raw_text` in Postgres — not from logs.
- The `LOG_PROMPTS` validator ensures a misconfigured production environment fails at startup rather than silently logging PII.
## Revisit Trigger
If a redacted prompt logging capability is needed (e.g., logging only structure/schema without field values), implement a dedicated `PromptRedactor` utility that strips values and logs only field names and token counts. This is a future enhancement, not an MVP requirement.