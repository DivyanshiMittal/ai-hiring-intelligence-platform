# ADR-005 — SSE over WebSockets for Evaluation Streaming

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-5 |

## Context
Answer evaluation and job progress must stream token-by-token to the browser for perceived latency. The streaming protocol choice affects proxy compatibility, server resource consumption, and connection management.

## Decision
**Server-Sent Events (SSE)** for all server-to-client streaming. The browser never sends data over the stream channel; answers are submitted via standard `POST` requests.

## Rationale
- SSE is unidirectional (server → client), which is exactly the use case. Bidirectional channel is not needed.
- SSE is plain HTTP/1.1 chunked transfer — works through every proxy, load balancer, and CDN with correct headers (see ADR-012).
- SSE has automatic reconnect built into the browser's `EventSource` API.
- SSE connections are stateless per connection: the interview session state lives in the Postgres checkpointer (LangGraph), so any worker can resume any session after a reconnect (no connection affinity required).
- WebSockets require upgrade handshake, custom reconnect logic, proxy configuration, and connection affinity considerations.

## Rejected Alternative
WebSockets: bidirectional channel not needed; connection affinity complicates horizontal scaling; proxy configuration in staging/production is error-prone.

## Consequences
- SSE is unidirectional only. If a future feature requires bidirectional real-time communication (e.g., voice interviews), this decision must be revisited.
- SSE requires specific response headers to prevent buffering (see ADR-012).
- The BFF must proxy SSE streams using the Node.js runtime (see ADR-012).

## Revisit Trigger
A bidirectional or interactive feature appears — specifically voice interviews (TDD §11, enhancement #2).
