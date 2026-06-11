# ADR-022 — Structured JSON Logging via structlog and pino
 
| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Optional Issue O-6, TDD NFR-6 |
 
## Context
OpenTelemetry is specified for traces and metrics. Log format was unspecified. Mixed formats (some JSON, some plaintext) make log aggregation, alerting, and trace-log correlation painful and are very difficult to retrofit across a codebase.
 
## Decision
All services emit **structured JSON logs to stdout**. The orchestrator (Docker / Kubernetes) handles collection.
 
### Python (FastAPI + ARQ Worker)
**`structlog`** with OTel trace ID injection:
 
```python
# app/core/logging.py
import structlog
from opentelemetry import trace
 
def add_trace_context(logger, method, event_dict):
    span = trace.get_current_span()
    if span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict
 
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        add_trace_context,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    logger_factory=structlog.PrintLoggerFactory(),
)
```
 
Every log line is a JSON object on a single line to stdout. Example:
```json
{"event": "resume_parse_started", "job_id": "...", "trace_id": "...", "span_id": "...", "level": "info", "timestamp": "2026-06-10T12:00:00Z"}
```
 
### TypeScript (Next.js BFF)
**`pino`** in JSON mode:
 
```typescript
// lib/logger.ts
import pino from 'pino';
export const logger = pino({ level: 'info' });
// In development, use pino-pretty for human-readable output
```
 
### Standard Fields
Every log line MUST include: `timestamp`, `level`, `event` (the message), `service` (e.g., `api`, `worker`, `web`), `trace_id` (when available), `span_id` (when available).
 
### Log Levels
- `DEBUG`: development only; disabled in production via `LOG_LEVEL=info` env.
- `INFO`: normal operational events (job started, job completed, cache hit).
- `WARNING`: recoverable unexpected events (retry triggered, schema repair attempted).
- `ERROR`: unrecoverable failures (job failed, DB connection lost).
## Consequences
- `print()` statements in application code are forbidden (lint rule via `ruff`). Use `structlog.get_logger()`.
- Log aggregators (e.g., Loki, Datadog, CloudWatch) can parse JSON natively with no transformation.
- Trace-log correlation works automatically: a log line's `trace_id` links directly to the corresponding OTel trace.
## Revisit Trigger
If the platform adopts a log management service that requires a specific format (e.g., Datadog's `dd.trace_id` convention), add a processor to `structlog` that maps field names. The underlying JSON-to-stdout approach does not change.
 