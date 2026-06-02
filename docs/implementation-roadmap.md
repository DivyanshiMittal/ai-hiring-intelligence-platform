# Implementation Roadmap — Hiring-Copilot

| | |
|---|---|
| **Project** | Hiring-Copilot — AI Hiring Intelligence Platform |
| **Document** | Implementation Roadmap |
| **Version** | 1.0 |
| **Companions** | [`architecture.md`](architecture.md) · [`TDD.md`](TDD.md) · [`interview-prep.md`](interview-prep.md) |

> **How to read this.** The project is decomposed into 10 sequential phases, each a **demoable vertical slice** — a deliberate choice so the project is never in a broken "half-built" state and every phase is independently showable in an interview. Time estimates assume one engineer working part-time (evenings/weekends); compress by ~40% for full-time. Each phase lists an explicit **Interview Value** — the specific competency it lets you demonstrate to a senior reviewer.

### Phase dependency graph

```
P1 Foundation
   └─> P2 Authentication
          └─> P3 Resume Upload
                 └─> P4 Resume Parsing
                        └─> P5 Qdrant Integration ──┐
                        └─> P6 JD Analysis ─────────┤
                                                    └─> P7 Skill Gap Engine
                                                           └─> P8 Question Generation
                                                                  └─> P9 LangGraph Agents
                                                                         └─> P10 Analytics Dashboard
```

### At-a-glance

| # | Phase | Est. Time | Headline deliverable |
|---|---|---|---|
| 1 | Foundation | 1 week | Monorepo + Docker Compose stack boots with one command |
| 2 | Authentication | 4–5 days | NextAuth session → signed service token → protected FastAPI |
| 3 | Resume Upload | 3–4 days | Async upload → object store → job + SSE progress |
| 4 | Resume Parsing | 1 week | LLM extraction → schema-validated structured profile |
| 5 | Qdrant Integration | 4–5 days | Section-aware chunking → embeddings → hybrid retrieval |
| 6 | JD Analysis | 4–5 days | JD → weighted requirements + match score breakdown |
| 7 | Skill Gap Engine | 4–5 days | RAG-grounded gaps with severity ranking |
| 8 | Question Generation | 4–5 days | Gap-driven questions with scoring rubrics |
| 9 | LangGraph Agents | 1.5 weeks | Checkpointed adaptive interview + streamed evaluation |
| 10 | Analytics Dashboard | 4–5 days | Readiness index, trends, materialized-view aggregates |

**Total estimate:** ~7–8 weeks part-time (~4–5 weeks full-time).

---

## Phase 1 — Foundation

**Objectives**
- Stand up the monorepo skeleton (`apps/web`, `apps/api`, `packages/`, `infra/`) with enforced domain boundaries.
- Make the full local stack reproducible: one command brings up web + api + worker + PostgreSQL + Qdrant + Redis + MinIO.
- Wire the cross-cutting plumbing that every later phase depends on: config, DB connection, migrations, telemetry, error envelope, CI.

**Deliverables**
- Docker Compose stack (all backing services) + three service Dockerfiles (`web`, `api`, `worker`).
- Alembic baseline migration; PostgreSQL initialized; `skills_taxonomy` seed script (can be stubbed).
- FastAPI app with `/healthz` + `/readyz`, structured error envelope, OpenTelemetry scaffolding.
- Next.js 15 app (App Router) with Tailwind + shadcn/ui installed and a placeholder dashboard shell.
- OpenAPI → TypeScript type generation pipeline.
- CI running lint, type-check, and a smoke test.

**Success Criteria**
- `docker compose up` boots the entire stack cleanly on a fresh machine; `/readyz` reports all dependencies reachable.
- A trivial typed endpoint round-trips from the Next.js client through the BFF to FastAPI and back.
- CI is green on an empty-but-wired repo.

**Estimated Time:** ~1 week

**Risks**
- *Docker networking / service-discovery friction* across 7 containers (mitigate: pin versions, healthcheck-gated `depends_on`).
- *Monorepo tooling sprawl* (mitigate: pick Turborepo + pnpm and uv/poetry early; don't bikeshed).
- *Over-scaffolding* — building abstractions before there's code to abstract (mitigate: minimum viable plumbing only).

**Interview Value**
Demonstrates **system setup & DevEx maturity** — reviewers judge a lot from "does `docker compose up` just work?" Shows you think about reproducibility, healthchecks, and CI from day one, not as an afterthought.

---

## Phase 2 — Authentication

**Objectives**
- Establish identity end-to-end: NextAuth session in the browser → short-lived signed service token → verified by FastAPI.
- Make FastAPI private and trust-no-client: the API authorizes via an actor context, never client-supplied identity.

**Deliverables**
- NextAuth configured (OAuth provider + email) with session handling in the BFF.
- BFF mints a short-lived (~5 min) asymmetric-signed service JWT per request to call FastAPI.
- FastAPI dependency that verifies the token and produces an `actor` context.
- `users` table + repository/actor authorization layer (the seam for row-level scoping).
- Protected route + protected endpoint; unauthenticated requests rejected.

**Success Criteria**
- An unauthenticated request to a protected endpoint returns `401`; an authenticated one succeeds with the correct `user_id` in scope.
- The Gemini key and service-signing key never appear in any client bundle.
- A tampered/expired service token is rejected by FastAPI.

**Estimated Time:** ~4–5 days

**Risks**
- *NextAuth ↔ FastAPI token bridging* is fiddly (mitigate: settle the JWT signing/verification contract first, test in isolation).
- *Authorization sprawl* if `user_id` checks leak into handlers (mitigate: enforce it in the repository layer from the start — this is hard to retrofit).
- *Clock skew* breaking short-lived tokens (mitigate: small leeway, NTP-synced containers).

**Interview Value**
Demonstrates **security architecture** — the BFF-as-token-broker pattern, defense-in-depth (private API + actor-scoped authz), and "treat the client as untrusted." This is exactly where staff interviewers probe, and most portfolio projects get it wrong.

---

## Phase 3 — Resume Upload

**Objectives**
- Implement the async-first ingestion entry point: validated upload → object store → tracked job → live progress.
- Establish the `processing_jobs` + SSE pattern that every long-running AI operation will reuse.

**Deliverables**
- `POST /resumes/upload` with MIME-sniff + size + page-count validation, returning `202 { resume_id, job_id }`.
- Presigned-URL (or proxied) upload to S3/MinIO; `documents` + `resumes(PENDING)` rows.
- `processing_jobs` table + ARQ worker scaffold (parse task stubbed for now).
- `GET /jobs/{id}` poll + `GET /jobs/{id}/stream` SSE endpoint.
- Frontend upload UI with a live progress indicator consuming SSE.

**Success Criteria**
- Uploading a valid PDF returns in < 500 ms with a job that visibly transitions `queued → running → succeeded`.
- An oversized / wrong-type file is rejected *before* any storage write, with a typed error.
- Closing and reopening the page reflects the correct, persisted job status.

**Estimated Time:** ~3–4 days

**Risks**
- *Large-file handling / streaming uploads* (mitigate: presigned direct-to-storage uploads, caps enforced).
- *SSE through dev proxies* dropping connections (mitigate: heartbeat events, auto-reconnect on the client).
- *Validation by extension instead of content* (mitigate: real MIME sniffing — also a security item).

**Interview Value**
Demonstrates **async-first API design** and the impedance-matching between synchronous HTTP and slow backend work. The `202 + job + SSE` pattern is the backbone of the whole system and a strong talking point.

---

## Phase 4 — Resume Parsing

**Objectives**
- Turn raw resume text into a validated structured profile using Gemini behind the LLM gateway.
- Prove reliability from an unreliable model: structured output + schema validation + bounded self-repair.

**Deliverables**
- Document text extraction (PDF/DOCX).
- LLM gateway: Gemini client with structured-output enforcement, token accounting, `llm_audit_log` writes, retry/backoff.
- Parsing flow: extract → classify sections → extract structured → pydantic-validate → repair (≤2) → persist.
- `resumes.parsed_json` + normalized `resume_skills` (taxonomy mapping; unmapped → `raw_mention` + curation queue).
- Frontend structured-profile view.

**Success Criteria**
- ≥ 90% field-level accuracy on identity + skills across a golden set of resumes.
- Every parsed resume produces schema-valid JSON; a deliberately malformed LLM response triggers repair and either recovers or fails the job cleanly.
- `llm_audit_log` records tokens/cost/latency for every call; PII is redacted from logs.

**Estimated Time:** ~1 week

**Risks**
- *LLM output variability / invalid JSON* (mitigate: function-calling/JSON-schema mode + repair loop — the core defense).
- *Resume format diversity* (columns, tables, scans) (mitigate: focus golden set on common formats; flag low-confidence parses).
- *Cost creep on large resumes* (mitigate: page caps, per-request token budget).

**Interview Value**
Demonstrates **GenAI engineering rigor** — structured output, self-repair, prompt-as-code, cost/observability per call, and the gateway abstraction. This is the "do you actually understand productionizing LLMs?" phase.

---

## Phase 5 — Qdrant Integration

**Objectives**
- Make resumes (and later JDs) retrievable: section-aware chunking → embeddings → Qdrant collections.
- Establish hybrid retrieval and the truth-store/index discipline (outbox + rebuildability).

**Deliverables**
- Embedding pipeline (batched, content-checksum cached) running in workers.
- Qdrant collections (`resume_chunks`, `skills_taxonomy`, `learning_resources`, `interview_bank`) with HNSW config + payload indexes.
- Outbox pattern: Postgres write → outbox → worker upsert to Qdrant; **server-injected `user_id` filter** on all queries.
- Hybrid retriever (dense + BM25, fused with RRF) with optional rerank.
- A documented **reindex-from-Postgres** job proving Qdrant is rebuildable.

**Success Criteria**
- A parsed resume's chunks are queryable within the staleness SLO (< 10 s p95) after parsing.
- Hybrid retrieval measurably beats pure-vector on a small labeled recall@k set.
- Wiping a Qdrant collection and running the reindex job fully restores retrieval — no data loss.
- A query without a `user_id` filter is impossible to issue (enforced server-side).

**Estimated Time:** ~4–5 days

**Risks**
- *Chunking strategy quality* (mitigate: section-aware chunks; A/B against fixed-window on the recall set).
- *Embedding-model versioning* mistakes mixing spaces (mitigate: `model_version` per point; alias-swap upgrade path).
- *Tenant leakage* via missing filters (mitigate: centralize the filter injection; test cross-tenant isolation explicitly).

**Interview Value**
Demonstrates **RAG and vector-DB design** depth — hybrid retrieval, chunking strategy, multi-tenant isolation, and the all-important "the index is derived and rebuildable" invariant that separates real RAG systems from demos.

---

## Phase 6 — JD Analysis

**Objectives**
- Parse and normalize job descriptions into weighted requirements.
- Produce the explainable resume↔JD match score with its three-way breakdown.

**Deliverables**
- `POST /job-descriptions` (file or pasted text) → parsing → `jd_requirements` (must/nice + weight + min_years).
- `jd_chunks` embedded into Qdrant.
- `POST /matches`: parallel scorers (semantic / skill-coverage / experience) → weighted aggregate → `match_results` with `score_breakdown`.
- Frontend match view: overall score + sub-scores + matched-skill evidence.

**Success Criteria**
- A pasted JD yields a normalized, weighted requirement list mapped to canonical skills.
- Match scoring returns an `overall_score` with a transparent breakdown; re-running on identical inputs/model version is stable within ±1 point.
- The match response surfaces *why* — matched skills and their weight contributions.

**Estimated Time:** ~4–5 days

**Risks**
- *Score calibration / comparability across JDs* (mitigate: rubric anchoring; track via eval harness — flagged as an open risk in the TDD).
- *Weighting subjectivity* (mitigate: make weights explicit and tunable, not hidden in a prompt).
- *Determinism* of LLM-assisted scoring (mitigate: low temperature + structured output + range validation).

**Interview Value**
Demonstrates **explainable AI as a product requirement** and **parallel agent orchestration** (independent scorers → reduce). "How do you make an LLM score trustworthy and reproducible?" is a classic senior question this phase answers concretely.

---

## Phase 7 — Skill Gap Engine

**Objectives**
- Convert unmet JD requirements into actionable, RAG-grounded gaps with severity.

**Deliverables**
- Gap analysis over match results: `skill_gaps` rows with `gap_type` (missing / insufficient_depth / outdated), severity, required vs. current level.
- RAG grounding: each recommendation cites retrieved nodes (skills taxonomy / resources); generation forbids out-of-set claims.
- Severity ranking = severity × requirement weight.
- Frontend gap view, ordered and grouped.

**Success Criteria**
- Every gap references a retrievable source node (grounding is verifiable, not asserted).
- Gaps are correctly ordered by severity × weight.
- Hallucinated recommendations are caught by the grounding-validation check on the golden set.

**Estimated Time:** ~4–5 days

**Risks**
- *Hallucinated prerequisites* (mitigate: strict grounding contract + validation; the whole point of RAG here).
- *Taxonomy coverage gaps* producing low-quality recommendations (mitigate: graceful degradation + curation queue from Phase 4/5).
- *Over/under-flagging* gaps (mitigate: tune thresholds against labeled examples).

**Interview Value**
Demonstrates **grounded generation** — the difference between "the model said so" and "here's the cited source." Anti-hallucination design is a top concern at product companies shipping AI features.

---

## Phase 8 — Question Generation

**Objectives**
- Generate a personalized interview from gaps, JD must-haves, and resume claims — each question carrying a scoring rubric.

**Deliverables**
- `POST /interviews` → question set with `mode` (technical/behavioral/mixed) + `difficulty`.
- `interview_questions` rows with `rubric_json`, `competency`, `generated_from` provenance, `parent_question_id` (for later follow-ups).
- Few-shot grounding from `interview_bank`.
- Frontend interview-setup + question-preview UI.

**Success Criteria**
- ≥ 70% of generated questions map to a real gap or must-have (provenance is recorded and checkable).
- Every question has a non-empty, usable rubric.
- Difficulty and mode selections meaningfully change the generated set.

**Estimated Time:** ~4–5 days

**Risks**
- *Generic / off-target questions* (mitigate: ground in gaps + JD; record provenance to measure relevance).
- *Rubric quality* driving downstream eval (mitigate: rubric is part of generation output, validated for completeness).
- *Difficulty consistency* (mitigate: anchor difficulty with examples).

**Interview Value**
Demonstrates **prompt engineering for structured, personalized generation** and closing the loop from analysis → action. Shows the system *uses* its own gap analysis rather than producing it in isolation.

---

## Phase 9 — LangGraph Agents

**Objectives**
- Build the flagship agentic feature: a stateful, checkpointed, adaptive mock-interview session with streamed rubric-based evaluation.

**Deliverables**
- Interview Session graph (cyclic state machine): generate question → await answer → evaluate → conditional follow-up → summarize.
- **Postgres checkpointer** so sessions survive disconnects and any worker can resume any session.
- `POST /interviews/{id}/answers` streaming evaluation via SSE → `answer_evaluations` (dimension scores, feedback, strengths, improvements).
- Adaptive follow-up logic (weak answer on a competency → deeper probe), with hard max-iteration caps.
- Retrofit the Parsing (P4), Match/Gap (P6–7) flows into formal LangGraph graphs where it adds value.
- Frontend live interview chat with streamed feedback + disconnect/resume.

**Success Criteria**
- Disconnecting mid-interview and reloading restores exact session state from the checkpoint.
- Evaluation streams with first token < 2 s p95; scores are stable for identical answer + rubric + model version.
- Follow-ups fire only on weak competency answers; no loop exceeds its iteration cap.

**Estimated Time:** ~1.5 weeks

**Risks**
- *Checkpointer complexity / state-shape churn* (mitigate: freeze the state schema early; version it).
- *Runaway loops / cost* (mitigate: hard caps + token budgets enforced in the shared runtime).
- *Streaming + checkpointing interaction* edge cases (mitigate: test reconnect mid-stream explicitly).

**Interview Value**
Demonstrates **agentic AI** — the highest-signal phase. Cyclic graphs, checkpointed/resumable state, adaptive behavior, and the "resumability + statelessness" dual win. This is the centerpiece you'll spend the most interview time on.

---

## Phase 10 — Analytics Dashboard

**Objectives**
- Surface insight across all activity: trends, performance, top gaps, and a composite readiness index — without hurting the write path.

**Deliverables**
- `analytics_events` (append-only, monthly-partitioned) emitted across flows.
- Materialized views for dashboard aggregates (refreshed on schedule, ≤ 60 s freshness).
- `GET /analytics/overview`: avg match score, interviews completed, avg interview score, top gap skills, score trend, readiness index.
- Frontend dashboard (charts, readiness index, trend lines) — read from snapshots, served via RSC.

**Success Criteria**
- Dashboard figures reflect events within the freshness SLO and are strictly user-scoped.
- Analytics queries hit materialized views / replicas — a heavy dashboard load never contends with interview writes.
- Readiness index visibly responds to completed interviews and closed gaps.

**Estimated Time:** ~4–5 days

**Risks**
- *Aggregation load on the primary* (mitigate: materialized views + replicas — the explicit design).
- *Readiness-index definition* being arbitrary (mitigate: document the formula; make it explainable).
- *Event-schema drift* (mitigate: typed event contracts; version `properties`).

**Interview Value**
Demonstrates **read/write separation, event modeling, and analytics scalability** — CQRS-adjacent thinking (append-only events → materialized read models). Closes the product loop and shows you protect the hot path from reporting load.

---

## Cross-phase practices (apply throughout)

- **Eval harness from Phase 4 onward** — a golden-set regression gate in CI on every prompt/model change. Prompts are code.
- **`model_version` stamped** on every AI artifact from the first LLM call.
- **Demo checkpoint each phase** — record a 60-second walkthrough; these become your portfolio reel and interview talking points.
- **ADR per contested decision** — capture the "why" while it's fresh (e.g., SSE vs WebSockets, monolith vs services).
- **Observability is not a phase** — traces, audit log, and metrics are wired in Phase 1 and maintained continuously.

---

*See [`TDD.md`](TDD.md) for the requirement IDs each deliverable satisfies, and [`architecture.md`](architecture.md) for the structural detail behind each component.*
