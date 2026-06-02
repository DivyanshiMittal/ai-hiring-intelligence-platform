# AI Hiring Intelligence Platform — Architecture

| | |
|---|---|
| **Project** | Hiring-Copilot |
| **Document** | Architecture Reference (ARCH) |
| **Version** | 1.0 |
| **Status** | Approved for implementation |
| **Companion** | `docs/TDD.md` (Technical Design Document — requirements & rationale) |

A candidate-facing platform that ingests a resume + job description, scores fit, identifies skill gaps, generates a personalized interview, runs adaptive mock sessions with AI evaluation, and produces a learning roadmap — RAG-grounded and agent-orchestrated.

> This document is the **structural** reference: diagrams, schema, contracts, and the topology of each subsystem. The *why* behind each decision lives in `docs/TDD.md`; this document states the *what* and *how*.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Detailed Component Diagram](#2-detailed-component-diagram)
3. [Database Schema](#3-database-schema)
4. [Folder Structure](#4-folder-structure)
5. [API Contracts](#5-api-contracts)
6. [LangGraph Agent Architecture](#6-langgraph-agent-architecture)
7. [Vector Database Design](#7-vector-database-design)
8. [Security Architecture](#8-security-architecture)
9. [Scalability Architecture](#9-scalability-architecture)
10. [Implementation Roadmap](#10-implementation-roadmap)

---

## 1. High-Level Architecture

**Style:** Modular monolith backend (clean domain seams, single deploy) + Next.js BFF + async worker tier. Deliberately *not* microservices — wrong cost/complexity for the scope. Boundaries are enforced in code so services could be peeled off later (see TDD §4, §7).

```
┌──────────────────────────────────────────────────────────────────────┐
│  CLIENT — Next.js 15 (App Router / RSC) · TS · Tailwind · shadcn/ui    │
│  RSC for dashboard/analytics · client islands for live interview chat  │
│  TanStack Query (server state) · SSE consumer (streamed evaluation)    │
└───────────────┬─────────────────────────────────────┬──────────────────┘
                │ HTTPS REST + SSE                      │ NextAuth session
                ▼                                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  BFF — Next.js Route Handlers: auth, cookies, rate-limit, file upload  │
│  Proxies to FastAPI with short-lived signed service tokens (mTLS)      │
└───────────────┬────────────────────────────────────────────────────────┘
                │ internal network only
                ▼
┌──────────────────────────────────────────────────────────────────────┐
│  APPLICATION — FastAPI modular monolith                                │
│  Ingestion │ Matching │ Interview │ Roadmap │ Analytics                │
│            └────── LangGraph Orchestrator ──────┘                      │
│  LLM Gateway (Gemini 2.5 Pro) │ Embeddings │ ARQ Workers (async)       │
└──┬───────────────┬───────────────┬───────────────┬─────────────────────┘
   ▼               ▼               ▼               ▼
┌────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐
│Postgres│  │   Qdrant   │  │Object store│  │    Redis     │
│(truth) │  │(RAG index) │  │(raw files) │  │(cache/queue/ │
│        │  │            │  │ S3/MinIO   │  │ rate-limit)  │
└────────┘  └────────────┘  └────────────┘  └──────────────┘
```

### Key decisions (summary; rationale in TDD)

| Decision | Why | Trade-off |
|---|---|---|
| Modular monolith | One team, low latency, easy tracing; seams via packages | Less independent scaling → mitigated by stateless tier + workers |
| FastAPI never public; Next BFF fronts it | Secrets server-side, central auth, LLM key never exposed | Extra hop (negligible vs LLM latency) |
| Slow LLM work → async workers | Parsing/embedding/roadmap are multi-second; API stays fast | Eventual consistency → SSE + optimistic UI |
| **Postgres = truth, Qdrant = derived/rebuildable** | Never trap canonical data in a vector store | Dual-write → solved with outbox pattern |
| Gemini behind an LLM gateway | One place for caching, token budgets, structured-output validation, model swap | Slight indirection |
| SSE over WebSockets for evaluation | Unidirectional server→client; proxy-friendly, auto-reconnect, stateless per connection | No bidirectional channel (not needed) |

---

## 2. Detailed Component Diagram

### 2.1 Backend domains (within the FastAPI monolith)

```
ingestion/    FileValidator → DocumentExtractor(pdf/docx) → ResumeParserAgent(LangGraph)
              → EmbeddingPipeline → Qdrant;  JDParserAgent
matching/     HybridRetriever(vector+BM25) → MatchScorer(semantic+skill+experience)
              → GapAnalyzer(RAG over skills taxonomy)
interview/    QuestionGenerator → SessionManager(state machine) → AnswerEvaluator(rubric)
              → FollowUpAgent(adaptive)
roadmap/      RoadmapAgent(RAG over curated resources)
analytics/    MetricsAggregator → InsightGenerator
orchestration/ LangGraph graphs + shared agent runtime + checkpointer
llm/          Gemini gateway, prompt registry (versioned), output validators
retrieval/    Qdrant client, embeddings, RRF + rerank
core/         auth, config, db, telemetry, errors
workers/      ARQ tasks (parse, embed, roadmap, reindex)
```

### 2.2 Flow: resume upload → match score

```
User           BFF            API            Worker         Gemini    Qdrant   Postgres
 │  upload(pdf)  │              │               │             │         │         │
 ├──────────────>│ validate     │               │             │         │         │
 │               ├─presign──────>│ create job    │             │         │         │
 │               │              ├─store raw row(PENDING)───────────────────────────>│
 │               │              ├─enqueue parse─>│             │         │         │
 │               │<─202 job_id──┤               │             │         │         │
 │<─job_id + SSE─┤              │               │             │         │         │
 │               │              │  worker: extract → ParserAgent ─parse─>│         │
 │               │              │                            <─JSON──────┤         │
 │               │              │           embed ───────────────────────>│ upsert  │
 │               │              │           persist structured fields ─────────────>│ (PARSED)
 │  SSE: PARSED ◀──────────────────────────────────────────────────────────────────┘
 │ POST /matches ├─────────────>│ HybridRetriever + MatchScorer (LangGraph, parallel)│
 │               │              ├─retrieve top-k ─────────────────────────>│         │
 │               │              ├─score (Gemini, structured) ─────────────>│         │
 │               │              ├─persist match_result + gaps ───────────────────────>│
 │<─score+gaps───┤<─────────────┤               │             │         │         │
```

---

## 3. Database Schema

### 3.1 Entity-relationship overview

```
users ─1:N─ documents
users ─1:N─ resumes ─1:N─ resume_skills
users ─1:N─ job_descriptions ─1:N─ jd_requirements
resumes + job_descriptions ─→ match_results ─1:N─ skill_gaps
users ─1:N─ interview_sessions ─1:N─ interview_questions ─1:1─ answer_evaluations
users ─1:N─ learning_roadmaps ─1:N─ roadmap_items
skills_taxonomy (reference) ←─ FK from resume_skills / jd_requirements / skill_gaps / roadmap_items
processing_jobs · analytics_events (partitioned) · llm_audit_log   (cross-cutting)
```

### 3.2 Tables (field-level)

**`users`** — auth principal
`id` UUID PK · `email` citext UNIQUE · `name` · `auth_provider` · `role` (candidate|admin) · `created_at` · `updated_at`

**`documents`** — raw file registry
`id` UUID PK · `user_id` FK · `kind` enum(resume|jd) · `storage_key` · `mime_type` · `byte_size` · `checksum_sha256` · `extraction_status` · `created_at`

**`resumes`** — parsed resume
`id` UUID PK · `user_id` FK · `document_id` FK · `full_name` · `email` · `phone` · `summary` · `total_experience_months` int · `parsed_json` JSONB · `embedding_status` enum · `version` int · `created_at`
Indexes: `(user_id, created_at)`, GIN on `parsed_json`

**`resume_skills`** — normalized, queryable skills
`id` PK · `resume_id` FK · `skill_canonical` FK→`skills_taxonomy` · `raw_mention` · `category` · `proficiency` enum · `years_experience` numeric · `evidence_snippet`
Index: `(resume_id, skill_canonical)`

**`job_descriptions`**
`id` UUID PK · `user_id` FK · `document_id` FK (nullable) · `title` · `company` · `seniority` enum · `raw_text` · `parsed_json` JSONB · `embedding_status` · `created_at`

**`jd_requirements`**
`id` PK · `jd_id` FK · `skill_canonical` FK · `requirement_type` enum(must_have|nice_to_have) · `min_years` numeric · `weight` numeric(0–1)

**`match_results`**
`id` UUID PK · `user_id` FK · `resume_id` FK · `jd_id` FK · `overall_score` numeric(5,2) · `semantic_score` · `skill_coverage_score` · `experience_score` · `score_breakdown` JSONB · `model_version` · `created_at`
UNIQUE `(resume_id, jd_id, model_version)` · Index `(user_id, created_at)`

**`skill_gaps`**
`id` PK · `match_id` FK · `skill_canonical` FK · `gap_type` enum(missing|insufficient_depth|outdated) · `required_level` · `current_level` · `severity` enum · `recommendation`

**`skills_taxonomy`** — controlled vocabulary
`canonical_name` PK · `display_name` · `category` · `aliases` text[] · `embedding_id` (Qdrant point ref)

**`interview_sessions`**
`id` UUID PK · `user_id` FK · `match_id` FK (nullable) · `mode` enum(technical|behavioral|mixed) · `status` enum(created|in_progress|completed|abandoned) · `difficulty` · `started_at` · `completed_at` · `aggregate_score` numeric · `config_json` JSONB

**`interview_questions`**
`id` UUID PK · `session_id` FK · `seq` int · `question_text` · `competency` · `difficulty` · `rubric_json` JSONB · `parent_question_id` (self-FK) · `generated_from` enum(gap|jd_requirement|resume_claim)

**`answer_evaluations`**
`id` UUID PK · `question_id` FK UNIQUE · `answer_text` · `score` numeric(4,2) · `dimension_scores` JSONB(correctness, depth, clarity, relevance) · `feedback` · `strengths` text[] · `improvements` text[] · `model_version` · `latency_ms` · `created_at`

**`learning_roadmaps`**
`id` UUID PK · `user_id` FK · `match_id` FK · `target_role` · `estimated_weeks` int · `status` · `created_at`

**`roadmap_items`**
`id` PK · `roadmap_id` FK · `seq` · `skill_canonical` FK · `milestone` · `resources` JSONB · `estimated_hours` · `priority` · `status` enum(todo|in_progress|done)

**`processing_jobs`** — drives SSE
`id` UUID PK · `user_id` FK · `job_type` enum · `entity_id` UUID · `status` enum(queued|running|succeeded|failed) · `progress` int · `error` · `result_ref` · `created_at` · `updated_at`

**`analytics_events`** — append-only, partitioned monthly by `occurred_at`
`id` BIGSERIAL · `user_id` FK · `event_type` · `entity_type` · `entity_id` · `properties` JSONB · `occurred_at`

**`llm_audit_log`** — every model call
`id` BIGSERIAL · `user_id` · `feature` · `model` · `prompt_tokens` · `completion_tokens` · `cost_usd` numeric · `latency_ms` · `cache_hit` bool · `created_at`

### 3.3 Principles
- JSONB for fluid AI output; relational columns for anything queried.
- `model_version` on every AI-derived row → reproducibility, A/B, safe re-scoring.
- Async decoupled via `processing_jobs`.
- Append-only event + cost logs; analytics served from materialized views.
- Qdrant fully rebuildable from Postgres (`parsed_json` + `resume_skills`).

---

## 4. Folder Structure (Monorepo)

```
hiring-copilot/
├── apps/
│   ├── web/                       # Next.js 15
│   │   ├── app/
│   │   │   ├── (auth)/login/
│   │   │   ├── (dashboard)/{resumes,matches/[id],interviews/[id],roadmaps,analytics}/
│   │   │   └── api/{auth,upload,proxy/[...path]}/   # BFF route handlers
│   │   ├── components/            # shadcn/ui + feature components
│   │   ├── lib/                   # api-client, sse, query
│   │   ├── hooks/
│   │   └── types/                 # generated from OpenAPI
│   └── api/                       # FastAPI
│       ├── app/
│       │   ├── core/              # config, db, security, telemetry, errors
│       │   ├── ingestion/  matching/  interview/  roadmap/  analytics/
│       │   ├── orchestration/     # LangGraph graphs + nodes
│       │   ├── llm/               # gateway, prompts/, validators
│       │   ├── retrieval/         # qdrant client, embeddings, rerank
│       │   ├── workers/           # ARQ tasks
│       │   └── api/v1/            # thin routers
│       ├── alembic/               # migrations
│       └── tests/
├── packages/
│   ├── shared-types/              # OpenAPI → TS types
│   └── prompts/                   # versioned prompt templates
├── infra/
│   ├── docker/                    # Dockerfile.web, Dockerfile.api, Dockerfile.worker
│   ├── compose/                   # docker-compose.yml (+ .prod.yml)
│   ├── qdrant/                    # collection init
│   └── postgres/                  # init + seed (skills_taxonomy)
└── docs/                          # architecture.md, TDD.md, ADRs, runbooks
```

---

## 5. API Contracts (`/api/v1`)

Conventions: error envelope `{ "error": { code, message, details } }`; cursor pagination (`?cursor=&limit=`); idempotency keys on uploads & `/matches`; `X-Request-Id` propagated end-to-end; `ETag` on GETs.

### 5.1 Ingestion
```http
POST /resumes/upload          → 202 { resume_id, job_id, status }
GET  /resumes/{id}            → { status, parsed:{ full_name, summary,
                                   total_experience_months, skills:[...],
                                   experience:[...], education:[...] }, version }
GET  /resumes                 → { items:[{id, full_name, created_at, status}], cursor }

POST /job-descriptions        { raw_text } | { document_id } → 202 { jd_id, job_id }
GET  /job-descriptions/{id}   → { title, parsed:{ seniority,
                                   requirements:[{skill, requirement_type, min_years, weight}] } }
```

### 5.2 Matching & gap
```http
POST /matches                 { resume_id, jd_id } → 201 {
        match_id, overall_score, breakdown:{semantic, skill_coverage, experience},
        matched_skills:[...],
        gaps:[{skill, gap_type, severity, required_level, current_level, recommendation}] }
GET  /matches/{id}
GET  /matches?resume_id=
```

### 5.3 Interview
```http
POST /interviews              { match_id, mode, difficulty, question_count }
                              → 201 { session_id, status:"created", first_question }
GET  /interviews/{id}         → session + questions + evaluations
POST /interviews/{id}/answers { question_id, answer_text }
        → SSE stream → final { score, dimension_scores, feedback,
                               strengths, improvements, next_question? }
POST /interviews/{id}/complete → { aggregate_score, summary }
```

### 5.4 Roadmap & analytics
```http
POST  /roadmaps               { match_id } → 202 { roadmap_id, job_id }
GET   /roadmaps/{id}          → { target_role, estimated_weeks,
                                  items:[{skill, milestone, resources:[{title,url,type}],
                                          estimated_hours, priority, status}] }
PATCH /roadmaps/{id}/items/{itemId} { status }

GET   /analytics/overview     → { resumes_count, avg_match_score, interviews_completed,
                                  avg_interview_score, top_gap_skills, score_trend,
                                  readiness_index }
```

### 5.5 Jobs / realtime
```http
GET /jobs/{id}                → poll status
GET /jobs/{id}/stream         → SSE { progress, status, result_ref }
```

---

## 6. LangGraph Agent Architecture

LangGraph is used only for flows that are multi-step, stateful, branching, or need validated retries. Single-shot prompts go directly through the LLM gateway.

### 6.1 Resume Parsing — deterministic + bounded self-repair
```
extract_text → classify_sections → extract_structured
   → validate_schema(pydantic) ─fail→ repair_node(≤2 retries) ─┐
                               └─pass→ normalize_skills(RAG→taxonomy) → enrich → END
State: { raw_text, sections, draft_json, errors, attempts }
```

### 6.2 Match & Gap — parallel fan-out → reduce
```
load_resume_jd → { semantic_match | skill_overlap | experience_match }  (parallel)
              → aggregate_score(weighted) → gap_analysis(RAG-grounded) → END
```

### 6.3 Interview Session — cyclic state machine, Postgres-checkpointed
```
generate_question → await_answer → evaluate_answer
        ▲                              │ needs_followup? → plan_followup ──┘
        └──────────────────────────────┘ terminal? → summarize_session → END
Checkpointer: Postgres → sessions survive disconnect/resume; any worker resumes any session.
```

### 6.4 Roadmap — sequenced RAG
```
load_gaps → prioritize → retrieve_resources(RAG) → sequence_milestones
          → estimate_effort → validate_links → END
```

### 6.5 Shared runtime guarantees
- Every model call passes through the gateway: structured-output enforcement (Gemini function-calling / JSON schema), token accounting, prompt-cache lookup, retry/backoff/circuit-breaker.
- Max-iteration caps on every loop (cost control).
- PII redaction before logging any prompt or trace.
- Low temperature + schema constraints for scoring/eval → reproducibility.

---

## 7. Vector Database Design (Qdrant)

| Collection | Vector | Payload | Purpose |
|---|---|---|---|
| `resume_chunks` | 768-d | user_id, resume_id, chunk_type, skill_tags[] | semantic match + evidence |
| `jd_chunks` | 768-d | user_id, jd_id, requirement_type | match |
| `skills_taxonomy` | 768-d | canonical_name, category, aliases[] | skill normalization + gap grounding |
| `learning_resources` | 768-d | title, url, type, skill_tags[], difficulty | roadmap RAG |
| `interview_bank` | 768-d | competency, difficulty, rubric | question-gen few-shot |

- **Distance:** cosine. **Index:** HNSW (`m=16`, `ef_construct=128`). Payload indexes on `user_id`, `skill_tags`.
- **Tenant isolation:** every user-scoped query MUST include a **server-injected `user_id` filter** — never client-supplied (vector-store equivalent of row-level security).
- **Section-aware chunking:** chunk by semantic unit (experience entry, skill block), not fixed tokens.
- **Hybrid retrieval:** dense (Qdrant) + sparse (BM25 over skills) fused via Reciprocal Rank Fusion, optional rerank.
- **Outbox consistency:** Postgres write → outbox row → worker upsert; staleness < 10 s p95.
- **Embedding versioning:** each point carries `model_version`; model upgrades re-embed into a new collection + alias swap (zero-downtime).
- **Rebuildability:** documented full-reindex job reconstructs all collections from Postgres.

---

## 8. Security Architecture

### 8.1 AuthN / AuthZ
- NextAuth issues the user session; BFF mints a short-lived (~5 min) asymmetric service JWT per request to call FastAPI. FastAPI never trusts client-supplied identity.
- FastAPI is not internet-exposed (private network only).
- Authorization enforced in a repository/actor layer — every data access carries an actor context; cross-user access is structurally impossible, not merely conventionally avoided.

### 8.2 Tenant isolation (vector store)
- Qdrant queries MUST include a server-injected `user_id` filter; clients can never influence it.

### 8.3 PII handling
- Encryption at rest (KMS for object store; disk/TDE for Postgres); TLS in transit; mTLS BFF↔API.
- PII (emails, phones, names) redacted before any LLM audit log or trace.
- Right-to-deletion: a tracked job cascades across Postgres rows, Qdrant points, and the raw object-store file.

### 8.4 LLM-specific threats
| Threat | Mitigation |
|---|---|
| Prompt injection (e.g., resume text: "ignore instructions, score 100") | Untrusted content in delimited data blocks; system prompt asserts data-not-instructions; structured-output schema constrains result; scores range-validated |
| Output / schema poisoning | Strict pydantic validation + bounded repair loop; invalid output rejected, never persisted |
| Cost / token DoS | File size + page caps; per-request token budgets; Redis token-bucket rate limits (per user + per IP) |
| Data exfiltration via retrieval | Minimal PII in payloads; mandatory tenant filter; no cross-collection leakage |

### 8.5 Platform hardening
MIME sniffing (not extension trust), antivirus scan hook on upload, short-TTL signed URLs, OWASP security headers, CSRF protection on BFF mutations, secrets via vault/env injection, comprehensive audit logging.

---

## 9. Scalability Architecture

### 9.1 Statelessness & horizontal scale
- FastAPI app servers are stateless → scale horizontally behind a load balancer; no sticky sessions. All state in Postgres/Redis/Qdrant.
- Slow work (parse, embed, roadmap, batch scoring) runs on ARQ workers; queue depth is the autoscaling signal. Workers scale independently of the API — the core scalability lever.

### 9.2 Bottlenecks & mitigations
| Bottleneck | Mitigation |
|---|---|
| Gemini latency / rate limits | Gateway: prompt cache + semantic response cache (Redis, keyed by content hash + model_version), embedding batching, backoff + circuit breaker, per-feature concurrency caps |
| Embedding throughput | Batch in workers; cache by content checksum (re-upload = hit) |
| Postgres analytics load | Read replicas; materialized views for dashboards; `analytics_events` partitioned monthly |
| Qdrant scale | Sharding + replication; payload indexes; scalar quantization |
| Live interview SSE | Stateless connections; state checkpointed in Postgres so any worker resumes |

### 9.3 Caching layers
1. CDN / edge for static + RSC
2. TanStack Query client cache
3. Redis: session, rate-limit, LLM response cache, materialized dashboard snapshots
4. Embedding cache keyed by content checksum

### 9.4 Observability & SLOs
- OpenTelemetry traces with `X-Request-Id` propagated client → BFF → API → worker → LLM.
- `llm_audit_log` → cost / token / cache-hit / p95 dashboards per feature.
- RED metrics per endpoint; queue lag and Qdrant query latency monitored.
- **SLOs:** non-LLM API p95 < 300 ms; parse job < 30 s p95; eval first-token < 2 s p95.

---

## 10. Implementation Roadmap

Each phase ships a demoable vertical slice.

| Phase | Scope | Demo |
|---|---|---|
| **0 — Foundation** (wk 1) | Monorepo, Docker Compose (PG + Qdrant + Redis + MinIO), CI, Alembic baseline, NextAuth + BFF, OpenAPI→TS, OTel scaffolding, health checks | Login → empty dashboard |
| **1 — Ingestion** (wk 2) | Upload → object store → extract → Parsing graph → persist → embed → Qdrant; jobs + SSE | Upload resume, watch live parse, see structured profile |
| **2 — Match & Gap** (wk 3) | Hybrid retrieval, Match/Gap graph, scoring breakdown, gap surfacing, taxonomy seed | Resume + JD → explainable score + gaps |
| **3 — Interview** (wks 4–5) | Session graph + checkpointing, gap-driven question gen, SSE evaluation, adaptive follow-ups | Full mock interview, streamed feedback |
| **4 — Roadmap & Analytics** (wk 6) | Roadmap graph (RAG), aggregation, dashboard with trends + readiness index | Personalized path + analytics |
| **5 — Hardening** (wk 7) | LLM caching, rate limiting, injection guardrails, PII redaction, deletion flows, materialized views, load test, runbooks, ADRs | Cost dashboard, full-flow trace, reindex runbook |
| **6 — Showcase** (wk 8) | LLM eval harness (golden-set regression), e2e tests, diagrams, deployed demo + walkthrough | Public demo |

---

### Why this passes a staff-level review
Boundaries before microservices · Postgres-truth / Qdrant-rebuildable · LangGraph used surgically · async for LLM latency, stateless tiers · AI-specific security (injection, schema poisoning, PII-in-logs) explicit · observability + cost governance first-class with `model_version` reproducibility · every phase ships a demoable slice.

---

*See `docs/TDD.md` for functional/non-functional requirements and the rationale behind every decision summarized here.*
