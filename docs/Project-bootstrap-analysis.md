# Project Bootstrap Analysis
 
| | |
|---|---|
| **Project** | Hiring-Copilot — AI Hiring Intelligence Platform |
| **Document** | Project Bootstrap Analysis |
| **Version** | 2.0 (updated post critical-issues resolution) |
| **Date** | 2026-06-10 |
| **Status** | Ready for Phase 1 implementation (pending final readiness assessment) |
| **Author** | Principal AI Architect / Technical Lead |
| **Audience** | Implementing engineers; staff reviewers |
 
> **Changelog from v1.0:** All 7 critical issues have been accepted and resolved. Architecture assumptions have been updated. ADR set is complete (ADR-001 through ADR-022). This v2.0 reflects the resolved state and is the authoritative pre-implementation baseline.
 
---
 
## 1. System Overview
 
Hiring-Copilot is a candidate-facing AI hiring intelligence platform. Its core loop: **parse → match → identify gaps → generate interview → evaluate answers → produce learning roadmap → report analytics.** Every step is RAG-grounded and agent-orchestrated.
 
**Stack summary:**
- Frontend: Next.js 15 (App Router, RSC), TypeScript, Tailwind, shadcn/ui, TanStack Query
- Backend: FastAPI modular monolith, Python 3.12, ARQ workers
- AI: Gemini 2.5 Pro behind an internal LLM gateway, LangGraph for stateful flows
- Storage: PostgreSQL (truth), Qdrant (derived/rebuildable), Redis, MinIO/S3
- Infra: Docker Compose, OpenTelemetry, GitHub Actions CI
---
 
## 2. Architecture Decisions Index
 
All decisions are recorded in `docs/ADRs/`. This is the complete index.
 
| ADR | Decision | Status | Source |
|---|---|---|---|
| ADR-001 | Modular monolith over microservices | Accepted | AD-1 |
| ADR-002 | Next.js BFF fronts all client traffic; FastAPI private | Accepted | AD-2 |
| ADR-003 | Async-first API: 202 + job_id for slow operations | Accepted | AD-3 |
| ADR-004 | LangGraph for stateful/branching flows only | Accepted | AD-4 |
| ADR-005 | SSE over WebSockets for evaluation streaming | Accepted | AD-5 |
| ADR-006 | All LLM calls via internal gateway | Accepted | AD-6 |
| ADR-007 | Worker uses API image with CMD override | **Accepted (C-1 resolution)** | C-1 |
| ADR-008 | BFF proxy: generic reverse proxy with envelope normalisation | Accepted | R-1 |
| ADR-009 | Idempotency key column contract | **Accepted (C-5 resolution)** | C-5 |
| ADR-010 | Skills taxonomy: MVT stub in P1, full ESCO in P4 | Accepted | R-3 |
| ADR-011 | Follow-up questions inserted on-the-fly during session | Accepted | R-10 |
| ADR-012 | BFF SSE proxy: Node.js runtime, headers, 15s heartbeat | **Accepted (C-4 resolution)** | C-4 |
| ADR-013 | UUIDv7 via application-layer `uuid7` package | **Accepted (C-3 resolution)** | C-3 |
| ADR-014 | ClamAV sidecar: scaffolded P1, activated P5 | Accepted | SEC-5 |
| ADR-015 | PostgreSQL extension bootstrap: init.sql + baseline migration | **Accepted (C-2 resolution)** | C-2 |
| ADR-016 | Qdrant tenant isolation via TenantScopedQdrantRepository | **Accepted (C-6 resolution)** | C-6 |
| ADR-017 | Processing jobs write-order invariant | **Accepted (C-7 resolution)** | C-7 |
| ADR-018 | model_version format: `provider:model_id:prompt_version` | Accepted | R-2 |
| ADR-019 | ARQ task handler idempotency pattern | Accepted | R-4 |
| ADR-020 | OTel trace propagation into ARQ workers | Accepted | R-6 |
| ADR-021 | LLM audit log PII redaction contract | Accepted | R-7 |
| ADR-022 | Structured JSON logging via structlog and pino | Accepted | O-6 |
 
---
 
## 3. Dependency Graph
 
```
[Infrastructure Layer]
  PostgreSQL 16 + citext + uuid-ossp + pg_stat_statements
  Redis 7
  Qdrant 1.9
  MinIO (S3-compatible)
  ClamAV (sidecar, activated Phase 5)
 
[Phase 1 — Foundation]  ← No dependencies; entry point
  Monorepo scaffold (pnpm + Turborepo + uv)
  Docker Compose (all 8 services including ClamAV stub)
  Dockerfile.web, Dockerfile.api (shared for worker)
  infra/postgres/init.sql (citext, uuid-ossp, pg_stat_statements)
  Alembic baseline migration (0001: extensions only)
  Alembic MVT taxonomy seed migration (0002)
  FastAPI app factory + core (config, db, errors, OTel, structlog)
  /healthz + /readyz endpoints
  Next.js 15 skeleton (App Router, Tailwind, shadcn/ui)
  OpenAPI → TypeScript type generation pipeline
  GitHub Actions CI (lint + type-check + smoke test)
 
[Phase 2 — Authentication]  ← requires Phase 1
  Alembic migration (0003: users table with citext email)
  NextAuth (OAuth + email/password)
  BFF service token minting (asymmetric JWT, ~5 min TTL)
  FastAPI JWT verification + ActorContext dependency
  Repository/actor authorization layer
  Protected routes and endpoints
 
[Phase 3 — Resume Upload]  ← requires Phase 2
  Alembic migration (0004: documents, resumes, processing_jobs with idempotency_key)
  POST /resumes/upload: MIME-sniff, size, page-count validation
  MinIO presigned upload
  processing_jobs table + ARQ scaffold
  Stale RUNNING jobs sweep task
  GET /jobs/{id} poll + GET /jobs/{id}/stream SSE (BFF proxy per ADR-012)
  Frontend upload UI + SSE consumer
 
[Phase 4 — Resume Parsing]  ← requires Phase 3
  PDF/DOCX text extraction (pdfminer + python-docx)
  LLM gateway implementation (ADR-006, ADR-021, ADR-018)
  Resume parsing LangGraph flow (extract → validate → repair → normalize)
  Alembic migration (0005: resume_skills, skills_taxonomy full + ESCO seed 0006)
  Taxonomy normalization service
  Frontend structured-profile view
  LLM eval harness: golden-set regression (CI gate from this phase forward)
 
[Phase 5 — Qdrant Integration]  ← requires Phase 4
  TenantScopedQdrantRepository (ADR-016)
  Embedding pipeline (batched, checksum-cached) in ARQ workers
  Qdrant collections init script: resume_chunks, jd_chunks, skills_taxonomy,
      learning_resources, interview_bank (HNSW m=16, ef=128; payload indexes)
  Outbox pattern for PG → Qdrant consistency
  Hybrid retriever (dense + BM25, RRF fusion, optional rerank)
  Reindex-from-Postgres runbook + job
  Cross-tenant isolation integration test (mandatory per ADR-016)
 
[Phase 6 — JD Analysis]  ← requires Phase 4 + Phase 5
  Alembic migration (0007: job_descriptions, jd_requirements, match_results + idempotency_key)
  POST /job-descriptions (file or pasted text)
  jd_chunks embedded into Qdrant
  POST /matches: parallel scorers LangGraph flow
  Frontend match view (score breakdown + evidence)
 
[Phase 7 — Skill Gap Engine]  ← requires Phase 5 + Phase 6
  Alembic migration (0008: skill_gaps)
  RAG-grounded gap analysis (retrieval from skills_taxonomy + learning_resources)
  Severity × weight ordering
  Frontend gap view
 
[Phase 8 — Question Generation]  ← requires Phase 7
  Alembic migration (0009: interview_sessions, interview_questions)
  POST /interviews: gap-driven question generation with rubrics
  few-shot grounding from interview_bank collection
  Frontend interview-setup + question-preview UI
 
[Phase 9 — LangGraph Agents]  ← requires Phase 8
  Interview session graph (cyclic, Postgres-checkpointed per ADR-R8)
  POST /interviews/{id}/answers: evaluation SSE stream
  On-the-fly follow-up insertion (ADR-011)
  Alembic migration (0010: answer_evaluations)
  Frontend live interview chat with streamed feedback + disconnect/resume
 
[Phase 10 — Analytics Dashboard]  ← requires Phases 3–9 (event sources)
  Alembic migration (0011: analytics_events partitioned, learning_roadmaps, roadmap_items)
  analytics_events emission across all flows
  Materialized views for dashboard aggregates (≤60s freshness)
  GET /analytics/overview
  POST /roadmaps + GET /roadmaps/{id} (roadmap LangGraph flow)
  Frontend dashboard (RSC, charts, readiness index)
```
 
---
 
## 4. Development Order Within Phase 1
 
The sequence within Phase 1 where ordering matters:
 
```
1. pnpm workspace + Turborepo + uv configuration
2. infra/docker/Dockerfile.api  (establishes the base; CMD = uvicorn)
3. infra/docker/Dockerfile.web
4. infra/postgres/init.sql  (citext, uuid-ossp, pg_stat_statements)
5. infra/compose/docker-compose.yml  (all 8 services, healthcheck-gated depends_on)
6. apps/api/app/core/config.py  (pydantic-settings; all env vars declared)
7. apps/api/app/core/uuidv7.py  (new_id() function; ADR-013)
8. apps/api/app/core/model_version.py  (ModelVersion value object; ADR-018)
9. apps/api/app/core/logging.py  (structlog setup; ADR-022)
10. apps/api/app/core/db.py  (async SQLAlchemy engine + session factory)
11. apps/api/app/core/errors.py  (error envelope + FastAPI exception handlers)
12. apps/api/app/core/telemetry.py  (OTel setup; trace propagation scaffolding)
13. apps/api/alembic/  (env.py, mako template, 0001_baseline.py)
14. apps/api/app/main.py  (FastAPI app factory, /healthz, /readyz)
15. apps/api/app/workers/main.py  (ARQ WorkerSettings scaffold)
16. infra/postgres/seed/mvt_skills.csv  (~500 skill records)
17. apps/api/alembic/versions/0002_seed_mvt_taxonomy.py
18. apps/web/  (Next.js 15 skeleton, Tailwind, shadcn/ui, placeholder dashboard)
19. packages/shared-types/  (openapi-typescript codegen pipeline)
20. .github/workflows/ci.yml  (lint + type-check + docker-compose smoke test)
```
 
---
 
## 5. Repository Blueprint
 
```
hiring-copilot/
│
├── .github/
│   └── workflows/
│       ├── ci.yml                    # lint + type-check + smoke (every commit)
│       └── eval.yml                  # LLM golden-set regression (prompt/model changes)
│
├── apps/
│   ├── web/                          # Next.js 15 BFF + frontend
│   │   ├── app/
│   │   │   ├── (auth)/login/page.tsx
│   │   │   ├── (dashboard)/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx          # dashboard home (RSC)
│   │   │   │   ├── resumes/page.tsx
│   │   │   │   ├── matches/[id]/page.tsx
│   │   │   │   ├── interviews/[id]/page.tsx
│   │   │   │   ├── roadmaps/page.tsx
│   │   │   │   └── analytics/page.tsx
│   │   │   └── api/
│   │   │       ├── auth/[...nextauth]/route.ts
│   │   │       ├── upload/route.ts         # typed upload handler (MIME + size gate)
│   │   │       ├── proxy/[...path]/route.ts # generic reverse proxy (ADR-008)
│   │   │       └── sse/
│   │   │           ├── jobs/[id]/route.ts  # SSE proxy, nodejs runtime (ADR-012)
│   │   │           └── interviews/[id]/answers/route.ts  # evaluation SSE proxy
│   │   ├── components/
│   │   │   ├── ui/                   # shadcn/ui primitives (auto-generated)
│   │   │   └── features/
│   │   │       ├── upload/
│   │   │       ├── resume/
│   │   │       ├── match/
│   │   │       ├── interview/
│   │   │       └── analytics/
│   │   ├── lib/
│   │   │   ├── api-client.ts         # typed fetch wrapper using shared-types
│   │   │   ├── service-token.ts      # server-side service JWT minting
│   │   │   ├── sse.ts                # useSSE hook with heartbeat handling
│   │   │   └── query.ts              # TanStack Query provider setup
│   │   ├── hooks/
│   │   ├── types/                    # DO NOT EDIT — generated from OpenAPI
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── api/                          # FastAPI modular monolith + ARQ workers
│       ├── app/
│       │   ├── main.py               # FastAPI app factory
│       │   ├── core/
│       │   │   ├── config.py         # pydantic-settings (all env vars)
│       │   │   ├── uuidv7.py         # new_id() — ADR-013
│       │   │   ├── model_version.py  # ModelVersion value object — ADR-018
│       │   │   ├── db.py             # async SQLAlchemy engine + session
│       │   │   ├── security.py       # JWT verification, ActorContext
│       │   │   ├── telemetry.py      # OTel setup + trace propagation — ADR-020
│       │   │   ├── logging.py        # structlog setup — ADR-022
│       │   │   ├── errors.py         # error envelope + exception handlers
│       │   │   └── dependencies.py   # FastAPI dependency providers
│       │   ├── ingestion/
│       │   │   ├── models.py         # SQLAlchemy ORM: Document, Resume, ResumeSkill
│       │   │   ├── schemas.py        # Pydantic I/O: UploadResponse, ResumeResponse
│       │   │   ├── repository.py     # DB access (actor-scoped)
│       │   │   ├── service.py        # business logic (write-order invariant — ADR-017)
│       │   │   └── router.py         # thin FastAPI router
│       │   ├── matching/
│       │   │   ├── models.py         # MatchResult, SkillGap
│       │   │   ├── schemas.py
│       │   │   ├── repository.py
│       │   │   ├── service.py
│       │   │   └── router.py
│       │   ├── interview/
│       │   │   ├── models.py         # InterviewSession, Question, AnswerEvaluation
│       │   │   ├── schemas.py
│       │   │   ├── repository.py
│       │   │   ├── service.py
│       │   │   └── router.py
│       │   ├── roadmap/
│       │   ├── analytics/
│       │   ├── orchestration/        # LangGraph graphs (Phase 4+)
│       │   │   ├── graphs/
│       │   │   │   ├── resume_parser.py
│       │   │   │   ├── match_scorer.py
│       │   │   │   ├── interview_session.py
│       │   │   │   └── roadmap_generator.py
│       │   │   ├── nodes/
│       │   │   └── checkpointer.py   # Postgres checkpointer setup (Phase 9)
│       │   ├── llm/
│       │   │   ├── gateway.py        # ADR-006, ADR-021
│       │   │   ├── prompts/          # versioned .jinja2 templates
│       │   │   │   └── v1/
│       │   │   └── validators.py     # Pydantic output schemas
│       │   ├── retrieval/
│       │   │   ├── qdrant_repository.py  # TenantScopedQdrantRepository — ADR-016
│       │   │   ├── shared_repository.py  # SharedQdrantRepository (reference collections)
│       │   │   ├── embeddings.py         # batch embedding pipeline
│       │   │   └── retriever.py          # hybrid retriever (dense + BM25 + RRF)
│       │   ├── workers/
│       │   │   ├── main.py               # ARQ WorkerSettings
│       │   │   └── tasks/
│       │   │       ├── parse_resume.py   # ADR-019 gate-then-work pattern
│       │   │       ├── parse_jd.py
│       │   │       ├── score_match.py
│       │   │       ├── generate_roadmap.py
│       │   │       ├── embed_document.py
│       │   │       ├── delete_user.py
│       │   │       └── sweep_stale_jobs.py  # ADR-019 stale RUNNING recovery
│       │   └── api/
│       │       └── v1/
│       │           └── router.py     # mounts all domain routers
│       ├── alembic/
│       │   ├── env.py
│       │   ├── script.py.mako
│       │   └── versions/
│       │       ├── 0001_baseline.py      # extensions only (C-2)
│       │       ├── 0002_seed_mvt_taxonomy.py  # ~500 skills (ADR-010)
│       │       ├── 0003_users.py         # Phase 2
│       │       ├── 0004_ingestion.py     # Phase 3: documents, resumes, processing_jobs
│       │       ├── 0005_resume_skills.py # Phase 4
│       │       ├── 0006_seed_esco.py     # Phase 4: full ESCO seed
│       │       ├── 0007_matching.py      # Phase 6: job_descriptions, match_results
│       │       ├── 0008_skill_gaps.py    # Phase 7
│       │       ├── 0009_interview.py     # Phase 8: sessions, questions
│       │       ├── 0010_evaluations.py   # Phase 9: answer_evaluations
│       │       └── 0011_analytics.py     # Phase 10: events, roadmaps
│       ├── tests/
│       │   ├── unit/
│       │   ├── integration/
│       │   │   └── test_qdrant_tenant_isolation.py  # mandatory — ADR-016
│       │   └── eval/
│       │       └── fixtures/             # golden-set JSON files (Phase 4+)
│       └── pyproject.toml
│
├── packages/
│   ├── shared-types/                 # openapi-typescript generated — DO NOT EDIT
│   └── prompts/
│       └── v1/                       # versioned prompt templates
│
├── infra/
│   ├── docker/
│   │   ├── Dockerfile.web
│   │   └── Dockerfile.api            # used for both api + worker (ADR-007)
│   ├── compose/
│   │   ├── docker-compose.yml        # local dev: all 8 services
│   │   └── docker-compose.prod.yml   # prod overrides
│   ├── postgres/
│   │   ├── init.sql                  # citext, uuid-ossp, pg_stat_statements (ADR-015)
│   │   └── seed/
│   │       ├── mvt_skills.csv        # ~500 tech skills (Phase 1 stub)
│   │       └── esco_skills.csv       # full ESCO export (Phase 4)
│   ├── qdrant/
│   │   └── init_collections.py       # collection creation script (Phase 5)
│   └── redis/
│       └── redis.conf
│
└── docs/
    ├── architecture.md               # source document (canonical; do not modify)
    ├── TDD.md                        # source document
    ├── roadmap.md                    # source document
    ├── critical-issues-resolution.md # this project's C-1 through C-7 resolutions
    ├── project-bootstrap-analysis.md # this document
    ├── project-journal.md            # running decision log
    ├── ADRs/
    │   ├── ADR-001 through ADR-022   # complete ADR set
    │   └── ADR-INDEX.md              # generated index
    └── runbooks/
        ├── reindex-qdrant.md         # Phase 5 deliverable
        ├── user-deletion.md          # Phase 5 deliverable (cascade order)
        └── stale-jobs-recovery.md    # Phase 3 deliverable
```
 
---
 
## 6. Updated Architecture Assumptions (Canonical Reference)
 
These assumptions **supersede** the source documents where they differ. They are the implementation baseline.
 
### Identity & PKs
- All PKs are UUIDv7, generated application-side via `uuid7.uuid7()` (ADR-013).
- No `server_default` PK generation anywhere in the schema.
### Docker & Infra
- Two application images: `Dockerfile.web`, `Dockerfile.api`.
- `api` image reused for `worker` via CMD override in `docker-compose.yml` (ADR-007).
- 8 containers in Compose: `web`, `api`, `worker`, `postgres`, `qdrant`, `redis`, `minio`, `clamav`.
- PostgreSQL `citext`, `uuid-ossp`, `pg_stat_statements` guaranteed by both `init.sql` and Alembic baseline migration (ADR-015).
### Schema Deltas vs Source Documents
- `documents` table: adds `idempotency_key VARCHAR(255)`, `UNIQUE NULLS NOT DISTINCT (user_id, idempotency_key)` (ADR-009).
- `match_results` table: adds same idempotency_key pattern (ADR-009).
- All AI-result tables: `model_version VARCHAR(128)` with check constraint `provider:model_id:prompt-name-vN` (ADR-018).
- `processing_jobs` table: includes `started_at TIMESTAMPTZ` (needed for stale-job sweep, ADR-019).
### API Contract Additions
- `POST /resumes/upload` and `POST /matches` accept optional `Idempotency-Key` header (ADR-009).
- All SSE routes use Node.js runtime, specific headers, 15s heartbeat (ADR-012).
- `X-Request-Id` generated at BFF if absent; propagated through all tiers including ARQ payloads (ADR-020).
### Security Additions
- All Qdrant access via `TenantScopedQdrantRepository`; direct Qdrant client import outside `app/retrieval/` is a lint error (ADR-016).
- LLM audit log never stores prompt or completion content (ADR-021).
- `LOG_PROMPTS=false` hardcoded in production config (ADR-021).
### Observability
- Structured JSON logging via `structlog` (Python) and `pino` (TypeScript) to stdout (ADR-022).
- OTel trace context propagated into ARQ via `_trace_context` payload field (ADR-020).
### Processing Jobs
- Write order: entity rows → `processing_jobs` → commit → ARQ enqueue (ADR-017).
- Worker gate pattern: `SELECT FOR UPDATE SKIP LOCKED WHERE status = QUEUED` (ADR-019).
- Stale RUNNING job sweep: cron every 5 minutes, resets jobs running > 10 minutes (ADR-019).
---
 
## 7. Risk Register (Residual, Post-Resolution)
 
All 7 critical issues are resolved. The following risks remain at Recommended or Optional severity.
 
| ID | Risk | Severity | Phase | Mitigation Status |
|---|---|---|---|---|
| R-1 | BFF proxy design | Resolved | P1 | ADR-008 |
| R-2 | model_version format | Resolved | P4 | ADR-018 |
| R-3 | Taxonomy seed gap | Resolved | P4 | ADR-010 |
| R-4 | ARQ idempotency | Resolved | P3 | ADR-019 |
| R-5 | Deletion cascade order | Documented | P5 | Runbook in P5 |
| R-6 | OTel worker propagation | Resolved | P1 | ADR-020 |
| R-7 | LLM PII redaction | Resolved | P4 | ADR-021 |
| R-8 | LangGraph checkpointer connections | Documented | P9 | ADR note; budget arithmetic in P9 |
| R-9 | Qdrant collection naming | Resolved | P5 | ADR-016 naming convention |
| R-10 | Follow-up write model | Resolved | P9 | ADR-011 |
| O-1 | Readiness index formula | Open | P10 | Define at Phase 10 start |
| O-2 | Golden-set fixture format | Documented | P4 | Tests/eval/fixtures/ structure |
| O-3 | Concurrent SSE rate limit | Open | P9 | Implement at Phase 9 |
| O-4 | Taxonomy seed in migrations | Resolved | P1 | ADR-010 |
| O-5 | RSC data fetching path | Documented | P1 | ADR-002 note on RSC |
| O-7 | Resume re-upload versioning | Documented | P3 | Covered in ingestion service |
 
---
 
## 8. Technology Versions (Pinned)
 
| Component | Version | Notes |
|---|---|---|
| Python | 3.12 | LTS; required for `match` syntax used in structlog |
| FastAPI | 0.115.x | Latest stable |
| SQLAlchemy | 2.0.x | Async-native |
| Alembic | 1.13.x | |
| ARQ | 0.26.x | Redis-based task queue |
| LangGraph | 0.2.x | Check compatibility with langgraph-checkpoint-postgres |
| pydantic | 2.x | Required for FastAPI 0.115+ |
| uuid7 | 0.1.x | Application-layer UUIDv7 (ADR-013) |
| structlog | 24.x | Structured logging (ADR-022) |
| opentelemetry-sdk | 1.x | OTel (ADR-020) |
| Next.js | 15.x | App Router + RSC |
| TypeScript | 5.x | |
| Tailwind CSS | 3.x | |
| TanStack Query | 5.x | |
| pino | 9.x | BFF structured logging (ADR-022) |
| PostgreSQL | 16 | Required for `NULLS NOT DISTINCT` (ADR-009) |
| Qdrant | 1.9.x | |
| Redis | 7.x | |
| MinIO | Latest stable | |
| ClamAV | latest (clamav/clamav image) | |
 
---
 
## 9. Phase 1 Success Criteria (Updated)
 
The following criteria define Phase 1 completion:
 
1. `docker compose up` from a clean machine brings all 8 services healthy within 60 seconds.
2. `/healthz` returns 200 on the FastAPI service.
3. `/readyz` returns 200 and confirms Postgres, Redis, Qdrant, and MinIO are reachable.
4. `alembic upgrade head` completes successfully against a fresh database, applying both the baseline migration (extensions) and the MVT taxonomy seed.
5. A smoke-test HTTP call through the Next.js BFF to a protected FastAPI endpoint returns 401 (unauthenticated) with the standard error envelope.
6. CI pipeline is green: lint (ruff, mypy, eslint, tsc), type-check, and the docker-compose smoke test all pass.
7. `docker run hiring-copilot-api arq app.workers.main.WorkerSettings` starts without error (worker CMD override verified).
---
 
*This document supersedes v1.0. It is the authoritative pre-implementation baseline. Implementation begins after the final readiness assessment confirms no remaining blockers.*
 