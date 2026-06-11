# ADR-012 — BFF SSE Proxy Implementation Contract

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-4 resolution, architecture.md AD-2 + AD-5 |

## Context
The BFF must proxy Server-Sent Events from the private FastAPI backend to the browser. Next.js Route Handlers have runtime-specific constraints for SSE. Without explicit specification, developers would use the Edge runtime (incompatible), omit anti-buffering headers, or open FastAPI directly from the browser.

## Decision

### Runtime
Every SSE route handler file MUST declare:
```typescript
export const runtime = 'nodejs';
```
The Edge runtime is **forbidden** for SSE routes. It does not support long-lived streaming responses or `ReadableStream` piping in the manner required.

### Required Response Headers
Every proxied SSE response MUST include:
```
Content-Type: text/event-stream
Cache-Control: no-cache, no-transform
X-Accel-Buffering: no
Connection: keep-alive
```
`X-Accel-Buffering: no` instructs nginx (and compatible CDNs) not to buffer the response. `no-transform` prevents gzip compression from buffering the stream. These are non-negotiable; omitting any one of them causes silent buffering in common infrastructure.

### Heartbeat
The BFF proxy MUST inject an SSE comment event (`:\n\n`) **every 15 seconds**. This is implemented as a `setInterval` in the `TransformStream`'s `transform` method. Heartbeats prevent upstream proxy timeouts (most proxies have a 30–60 s idle timeout) and tell the browser's `EventSource` the connection is alive.

FastAPI endpoints do **not** need to produce heartbeats. The BFF owns this responsibility.

### Implementation Pattern
```typescript
// Canonical pattern for all SSE proxy routes
export const runtime = 'nodejs';

export async function GET(request: NextRequest, { params }) {
  const serviceToken = await mintServiceToken();
  const upstreamUrl = `${process.env.API_INTERNAL_URL}/api/v1/jobs/${params.id}/stream`;

  const upstream = await fetch(upstreamUrl, {
    headers: {
      Authorization: `Bearer ${serviceToken}`,
      'X-Request-Id': request.headers.get('x-request-id') ?? crypto.randomUUID(),
    },
  });

  if (!upstream.ok || !upstream.body) {
    return new Response('Upstream SSE failed', { status: 502 });
  }

  let heartbeatInterval: ReturnType<typeof setInterval>;
  const { readable, writable } = new TransformStream();
  const writer = writable.getWriter();
  const encoder = new TextEncoder();

  heartbeatInterval = setInterval(() => {
    writer.write(encoder.encode(':\n\n')).catch(() => clearInterval(heartbeatInterval));
  }, 15_000);

  upstream.body
    .pipeTo(writable)
    .finally(() => clearInterval(heartbeatInterval));

  return new Response(readable, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache, no-transform',
      'X-Accel-Buffering': 'no',
      'Connection': 'keep-alive',
    },
  });
}
```

This pattern is used for:
- `GET /api/proxy/jobs/[id]/stream` → proxies `GET /api/v1/jobs/{id}/stream`
- `POST /api/proxy/interviews/[id]/answers` → proxies `POST /api/v1/interviews/{id}/answers` (evaluation stream)

### FastAPI SSE Endpoints
Private-network only. The browser never calls them directly. They produce standard SSE format (`data: {...}\n\n`). They do not produce heartbeats.

## Consequences
- SSE proxy routes require more code than the generic proxy (ADR-008). They are dedicated route handlers, not handled by the catch-all proxy.
- The 15-second heartbeat interval must be shorter than the shortest proxy/CDN idle timeout in any deployment environment. 15 s is conservative and safe for all common configurations.
- If Next.js changes its streaming API in a future version, this pattern may need updating. The contract (headers + heartbeat) does not change; only the implementation detail.

## Revisit Trigger
If a future Next.js version provides a cleaner SSE proxy abstraction, adopt it. The contract documented here remains the specification regardless of implementation.
