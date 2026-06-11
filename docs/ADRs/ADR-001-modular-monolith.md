# ADR-001 — Modular Monolith over Microservices

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-1 |

## Context
The platform spans five domains: ingestion, matching, interview, roadmap, analytics. The question is whether to deploy these as independent microservices or as a single deployable unit with enforced internal boundaries.

## Decision
A **modular monolith**: one FastAPI deployable with enforced domain packages (`ingestion/`, `matching/`, `interview/`, `roadmap/`, `analytics/`) plus a separate async worker tier. Domain boundaries are enforced via import rules in `pyproject.toml` — no domain imports another domain's internals; inter-domain communication uses published service interfaces.

## Rationale
- One team. Distributed transactions add coordination overhead with no benefit at this scale.
- Dominant latency is the LLM (~1–8 s), not service hops (~1 ms). Distribution buys nothing on the critical path.
- Shared SQLAlchemy session enables transactional writes across domains (e.g., `match_results` + `skill_gaps` in one transaction).
- Single deploy simplifies CI, rollback, and local development.
- Enforced package boundaries mean extraction to microservices is a structural refactor, not a rewrite.

## Rejected Alternative
Microservices from day one: distributed transaction complexity, network failure modes, polyglot dependency management overhead — none of these are justified by current load requirements.

## Consequences
- API and workers cannot scale independently per domain. Mitigated by the stateless API + worker-tier architecture (SCL-1) where the worker scales independently of the HTTP tier.
- A future microservice extraction of the interview engine is explicitly anticipated (TDD §11, enhancement #10).

## Revisit Trigger
A single domain's scaling requirements or deploy cadence genuinely diverges from the rest. Most likely candidate: interview engine under high concurrent session load.
