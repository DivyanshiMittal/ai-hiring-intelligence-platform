# ADR-020 — OpenTelemetry Trace Propagation into ARQ Workers
 
| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-6, TDD NFR-6, DEP-9 |
 
## Context
The architecture requires distributed tracing from client → BFF → API → worker → LLM via `X-Request-Id`. But ARQ communicates over Redis (not HTTP), so standard HTTP header propagation doesn't work at the API → worker boundary. Without explicit propagation, traces break at the worker, which is exactly where the LLM call (and thus the most valuable trace data) happens.
 
## Decision
 
### X-Request-Id Generation
The BFF generates `X-Request-Id` (UUIDv4) if the client does not supply one. It always forwards it to FastAPI. `X-Request-Id` maps 1:1 to the OTel `trace_id` for the HTTP legs.
 
### Propagation into ARQ
When an API service enqueues an ARQ task, it includes the current OTel trace context in the task payload:
 
```python
# In the service layer, after transaction commit
from opentelemetry import trace, propagate
from opentelemetry.propagators.textmap import DefaultTextMapPropagator
 
def _extract_trace_context() -> dict[str, str]:
    carrier: dict[str, str] = {}
    propagate.inject(carrier)  # injects 'traceparent', 'tracestate'
    return carrier
 
await arq_redis.enqueue_job(
    "parse_resume_task",
    job_id=str(job.id),
    _trace_context=_extract_trace_context(),  # ← propagated
)
```
 
### Restoration in Worker
Every ARQ task handler restores the trace context as its first action:
 
```python
from opentelemetry import propagate, trace
 
async def parse_resume_task(ctx: dict, job_id: str, _trace_context: dict | None = None) -> None:
    # Restore parent span from the enqueuing request
    parent_context = propagate.extract(_trace_context or {})
    tracer = trace.get_tracer(__name__)
 
    with tracer.start_as_current_span(
        "worker.parse_resume",
        context=parent_context,
        kind=trace.SpanKind.CONSUMER,
    ) as span:
        span.set_attribute("job.id", job_id)
        span.set_attribute("worker.queue", "default")
        # ... rest of handler
```
 
This creates a **linked span** (CONSUMER kind) that is a child of the original HTTP request's span. In Jaeger/Tempo, the full trace shows: HTTP request → API processing → worker pickup → LLM call, all under one trace ID.
 
### X-Request-Id in LLM Calls
The gateway passes `X-Request-Id` to Gemini via the `x-goog-request-id` header (or equivalent). It is also stamped on every `llm_audit_log` row via the active span's trace ID.
 
## Consequences
- All ARQ task function signatures MUST include `_trace_context: dict | None = None` as the last parameter. This is a convention enforced by code review.
- ARQ passes all kwargs to the task function, so `_trace_context` flows through without ARQ needing to understand it.
- If a task is enqueued without trace context (e.g., from a CLI script or stale sweep), `propagate.extract({})` returns an empty context — the worker creates a new root span. This is correct behaviour.
## Revisit Trigger
If ARQ adds native OTel support, adopt it and remove the manual `_trace_context` parameter pattern.