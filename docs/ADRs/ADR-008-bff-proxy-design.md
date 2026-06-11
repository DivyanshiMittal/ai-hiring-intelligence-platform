# ADR-008 — BFF Proxy: Generic Reverse Proxy with Envelope Normalisation

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-1 |

## Context
The BFF includes a catch-all proxy route (`app/api/proxy/[...path]/route.ts`) that forwards client requests to the private FastAPI backend. The question is whether this should be a typed-per-endpoint BFF or a generic reverse proxy.

## Decision
The proxy route is a **generic reverse proxy** with three responsibilities:
1. Inject the service JWT (`Authorization: Bearer <service_token>`) into every upstream request.
2. Propagate `X-Request-Id` (generating one if the client did not supply it).
3. Normalise error responses to the standard envelope `{ error: { code, message, details } }`.

The proxy does **not** validate request bodies or response shapes. Per-endpoint type safety is provided by OpenAPI-generated TypeScript types used at the call site in `lib/api-client.ts` — not by the proxy handler itself.

SSE routes are **not** handled by the generic proxy. They have dedicated route handlers that implement the SSE proxy contract (see ADR-012).

File uploads are **not** handled by the generic proxy. The `app/api/upload/route.ts` handler validates MIME type and size before forwarding to FastAPI (this is a security boundary, not a proxy).

## Rationale
- A typed-per-endpoint BFF duplicates every API contract in both the OpenAPI spec and the BFF route handler — two places to keep in sync.
- The OpenAPI-generated types in `packages/shared-types/` already provide compile-time safety at the call site.
- The generic proxy reduces the BFF surface area that engineers need to modify when adding new API endpoints.
- Security-sensitive routes (upload, SSE) get dedicated handlers with explicit validation.

## Consequences
- Adding a new FastAPI endpoint does not require a corresponding BFF route change — it is automatically proxied.
- The proxy does not enforce request schemas. A malformed request will be rejected by FastAPI's Pydantic validation and the error envelope will be normalised on the way back.
- Developers must not add business logic to the generic proxy handler.

## Revisit Trigger
A requirement emerges to validate or transform a specific endpoint's request/response at the BFF layer (e.g., for rate-limiting a specific field, or A/B testing response shapes). At that point, promote that endpoint to a dedicated typed route handler.
