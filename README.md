<div align="center">

# 🎯 Hiring-Copilot

### AI Hiring Intelligence Platform

**Upload a resume and a job description → get a fit score, skill-gap analysis, a personalized mock interview with AI evaluation, and a learning roadmap.**

RAG-grounded · Agent-orchestrated · Production-grade system design

[![Next.js](https://img.shields.io/badge/Next.js-15-black?logo=next.js)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript)](https://www.typescriptlang.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.11x-009688?logo=fastapi)](https://fastapi.tiangolo.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql)](https://www.postgresql.org/)
[![Qdrant](https://img.shields.io/badge/Qdrant-vector_db-DC244C)](https://qdrant.tech/)
[![Gemini](https://img.shields.io/badge/Gemini-2.5_Pro-4285F4?logo=google)](https://ai.google.dev/)
[![LangGraph](https://img.shields.io/badge/LangGraph-agents-1C3C3C)](https://langchain-ai.github.io/langgraph/)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker)](https://www.docker.com/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

</div>

---

## 📖 Project Overview

**Hiring-Copilot** is a candidate-facing platform that closes the loop between *"here's my resume"* and *"I'm ready for this role."* It ingests a resume and a target job description, then drives an end-to-end intelligence pipeline:

```
parse → match → identify gaps → generate a personalized interview
      → evaluate answers → produce a learning roadmap → report analytics
```

It is built as a **flagship engineering portfolio project** demonstrating full-stack engineering, GenAI engineering, RAG, agentic AI, and production system design — the kind of project meant to hold up under a staff-engineer review.

What makes it more than a wrapper around an LLM:

- **RAG-grounded** — gap recommendations and learning resources cite real, retrievable sources, not hallucinations.
- **Agent-orchestrated** — LangGraph drives stateful, self-correcting, checkpointed flows (the adaptive interview survives disconnects and resumes mid-session).
- **Truth-store discipline** — PostgreSQL is the source of truth; Qdrant is a derived, rebuildable index. No canonical data is ever trapped in the vector store.
- **Explainable by design** — every match score returns a breakdown and the evidence behind it; every answer evaluation is rubric-based.
- **Reproducible AI** — every AI-derived artifact is stamped with a `model_version`; upgrading a prompt or model never silently mutates past results.

> 📚 **Design docs:** [`docs/architecture.md`](docs/architecture.md) · [`docs/TDD.md`](docs/TDD.md) · [`docs/interview-prep.md`](docs/interview-prep.md)

---

## ✨ Features

| # | Feature | What it does |
|---|---|---|
| 1 | **Resume Upload** | PDF/DOCX upload with MIME-sniff validation; async parsing with live progress (SSE). |
| 2 | **Resume Parsing** | LangGraph extraction → structured profile (experience, education, skills with evidence), schema-validated with bounded self-repair. |
| 3 | **Job Description Upload** | Upload or paste a JD; parsed into title, seniority, and weighted `must_have` / `nice_to_have` requirements. |
| 4 | **Resume–JD Match Scoring** | 0–100 fit score decomposed into **semantic**, **skill-coverage**, and **experience** sub-scores — with a transparent, explainable breakdown. |
| 5 | **Skill Gap Analysis** | Per-requirement gaps (`missing` / `insufficient_depth` / `outdated`) with severity and RAG-grounded recommendations. |
| 6 | **Personalized Interview Generation** | Questions targeting your gaps, JD must-haves, and resume claims — each carrying a scoring rubric. |
| 7 | **Mock Interview Sessions** | Adaptive, stateful sessions with follow-up questions; checkpointed so they survive disconnects. |
| 8 | **Answer Evaluation** | Rubric-based scoring across correctness, depth, clarity, relevance — streamed token-by-token. |
| 9 | **Learning Roadmap Generation** | Ordered milestones with curated, link-validated resources (RAG) and effort estimates. |
| 10 | **Analytics Dashboard** | Match trends, interview performance, top gap skills, and a composite **readiness index**. |

---

## 🛠 Tech Stack

### Frontend
- **Next.js 15** (App Router, React Server Components)
- **TypeScript**
- **Tailwind CSS** + **shadcn/ui**
- **TanStack Query** (server state) · **SSE** consumer (streamed evaluation)

### Backend
- **FastAPI** (modular monolith) + **ARQ** async workers
- **LangGraph** — agent orchestration (parsing, matching, interview, roadmap graphs)
- **Gemini 2.5 Pro** — behind an internal LLM gateway (structured output, caching, cost governance)

### Data
- **PostgreSQL** — system of record (truth store)
- **Qdrant** — vector database / RAG retrieval (derived, rebuildable)
- **Redis** — cache, job queue, rate-limiting
- **Object storage (S3 / MinIO)** — raw uploaded files

### Platform
- **Docker** / **Docker Compose**
- **OpenTelemetry** — distributed tracing & observability
- **Alembic** — database migrations

---

## 🏗 Architecture

A **modular monolith** backend with a **Next.js BFF** and an **async worker tier** — deliberately not microservices (wrong cost/complexity for the scope), but with domain boundaries enforced in code so services can be extracted later.

```
┌──────────────────────────────────────────────────────────────────────┐
│  CLIENT — Next.js 15 (App Router / RSC) · TS · Tailwind · shadcn/ui    │
│  RSC for dashboard/analytics · client islands for live interview chat  │
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

**Key principles**

- **Async-first** — slow LLM work returns a `job_id` and runs on workers; the API stays fast (p95 < 300 ms for non-LLM endpoints).
- **Stateless tiers** — API scales on CPU, workers scale on queue depth, no sticky sessions.
- **Postgres = truth, Qdrant = derived** — the vector index is always rebuildable from Postgres.
- **LLM gateway** — one place for structured-output validation, caching, token budgets, and provider swap.

> Full diagrams, database schema, API contracts, the LangGraph agent graphs, and the vector-DB design live in [`docs/architecture.md`](docs/architecture.md).

---

## 📁 Folder Structure

```
hiring-copilot/
├── apps/
│   ├── web/                       # Next.js 15 (frontend + BFF)
│   │   ├── app/
│   │   │   ├── (auth)/login/
│   │   │   ├── (dashboard)/       # resumes, matches, interviews, roadmaps, analytics
│   │   │   └── api/               # BFF route handlers (auth, upload, proxy)
│   │   ├── components/            # shadcn/ui + feature components
│   │   ├── lib/                   # api-client, sse, query
│   │   ├── hooks/
│   │   └── types/                 # generated from OpenAPI
│   └── api/                       # FastAPI (backend)
│       ├── app/
│       │   ├── core/              # config, db, security, telemetry, errors
│       │   ├── ingestion/  matching/  interview/  roadmap/  analytics/
│       │   ├── orchestration/     # LangGraph graphs + nodes
│       │   ├── llm/               # Gemini gateway, prompts, validators
│       │   ├── retrieval/         # Qdrant client, embeddings, rerank
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
└── docs/                          # architecture.md, TDD.md, interview-prep.md, ADRs
```

---

## 🗺 Roadmap

Built as a sequence of demoable vertical slices.

- [ ] **Phase 0 — Foundation** · monorepo, Docker Compose (PG + Qdrant + Redis + MinIO), CI, Alembic baseline, NextAuth + BFF, OpenAPI→TS, OpenTelemetry
- [ ] **Phase 1 — Ingestion** · resume/JD upload → extract → parsing graph → embed → Qdrant; jobs + SSE
- [ ] **Phase 2 — Match & Gap** · hybrid retrieval, match/gap graph, explainable scoring, skills taxonomy
- [ ] **Phase 3 — Interview** · session graph + checkpointing, gap-driven question gen, streamed evaluation, adaptive follow-ups
- [ ] **Phase 4 — Roadmap & Analytics** · roadmap graph (RAG), aggregation, dashboard, readiness index
- [ ] **Phase 5 — Hardening** · LLM caching, rate limiting, prompt-injection guardrails, PII redaction, deletion flows, load test, runbooks
- [ ] **Phase 6 — Showcase** · LLM eval harness (golden-set regression), e2e tests, deployed demo

**Future enhancements:** multi-resume portfolios · voice mock interviews · recruiter/B2B mode · distilled answer-evaluation model · bias & fairness instrumentation. See [`docs/TDD.md`](docs/TDD.md) §11.

---

## 🚀 Getting Started

> ⚠️ **Placeholder** — implementation is in progress (see Roadmap). The steps below describe the intended developer experience; commands will be finalized as Phase 0 lands.

### Prerequisites

- **Docker** & **Docker Compose**
- **Node.js** 20+ and **pnpm** (frontend)
- **Python** 3.12+ and **uv** / **poetry** (backend)
- A **Google Gemini API key**

### 1. Clone

```bash
git clone https://github.com/<your-org>/hiring-copilot.git
cd hiring-copilot
```

### 2. Configure environment

```bash
cp .env.example .env
# Set at minimum:
#   GEMINI_API_KEY=...
#   POSTGRES_URL=...
#   QDRANT_URL=...
#   REDIS_URL=...
#   S3_ENDPOINT / S3_BUCKET / credentials
#   NEXTAUTH_SECRET=...
```

> 🔒 The Gemini key lives only in the API/worker tier — never expose it to the browser.

### 3. Start the stack

```bash
# Brings up: web + api + worker + PostgreSQL + Qdrant + Redis + MinIO
docker compose -f infra/compose/docker-compose.yml up
```

### 4. Initialize the database

```bash
# Run migrations + seed the skills taxonomy
docker compose exec api alembic upgrade head
docker compose exec api python -m app.scripts.seed_taxonomy
```

### 5. Open the app

| Service | URL |
|---|---|
| Web (Next.js) | http://localhost:3000 |
| API docs (FastAPI) | http://localhost:8000/docs |
| Qdrant dashboard | http://localhost:6333/dashboard |
| MinIO console | http://localhost:9001 |

### Running tests (planned)

```bash
pnpm test          # frontend
pytest             # backend unit + integration
pnpm test:e2e      # Playwright end-to-end
pytest -m eval     # LLM golden-set regression harness
```

---

## 🤝 Contributing

Contributions are welcome once Phase 0 lands. Until then, the design docs in [`docs/`](docs/) are the best place to understand intent and direction.

## 📄 License

[MIT](LICENSE)

---

<div align="center">

**Built to demonstrate production-grade GenAI system design.**

[Architecture](docs/architecture.md) · [Technical Design](docs/TDD.md) · [Staff-Review Q&A](docs/interview-prep.md)

</div>
