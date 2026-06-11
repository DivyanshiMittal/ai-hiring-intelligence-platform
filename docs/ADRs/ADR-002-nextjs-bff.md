# ADR-002 — Next.js BFF Fronts All Client Traffic; FastAPI is Private

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-2, TDD SEC-1 |

## Context
The platform has a FastAPI backend that holds the Gemini API key, service secrets, and all business logic. The question is whether the browser talks to FastAPI directly or through an intermediary.

## Decision
The browser communicates **only** with Next.js Route Handlers (the BFF layer). FastAPI is deployed on a private network with no public ingress. The BFF:
1. Manages the user session via NextAuth.
2. Mints a short-lived (~5 min) asymmetric-signed service JWT per request.
3. Injects the JWT into upstream calls to FastAPI.
4. Propagates `X-Request-Id`.
5. Normalises the error envelope for the frontend.
6. Proxies SSE streams (see ADR-012).

FastAPI verifies the service JWT on every request and trusts no identity claim from the client.

## Rationale
- The Gemini API key and service-signing private key never appear in the browser or any client bundle.
- Auth, rate-limiting, and CSRF protection are centralised in one place.
- RSC (React Server Components) can construct the service JWT server-side and call FastAPI directly without an extra HTTP hop through the BFF's route handlers — this is the correct RSC data-fetching pattern (see ADR-005 note on RSC).
- The extra hop for non-RSC requests is negligible vs LLM latency (~1–8 s).

## Rejected Alternative
Exposing FastAPI directly with CORS and browser-held JWT: leaks secrets, scatters auth, makes CSRF impossible to centralise.

## Consequences
- Every new endpoint requires a corresponding BFF proxy or RSC server action.
- BFF becomes a potential bottleneck; mitigated by its statelessness and Next.js horizontal scalability.

## Revisit Trigger
Never for the security posture. The hop cost will never be the bottleneck.
