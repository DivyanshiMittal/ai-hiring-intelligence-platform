# ADR-004 — LangGraph for Stateful/Branching Flows Only

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md AD-4 |

## Context
LangGraph is available for orchestrating multi-step AI workflows. The question is whether to use it universally or selectively.

## Decision
LangGraph is used **only for flows that require two or more of: persistent state, conditional branching, parallel fan-out, or validated self-repair loops**. Single-shot prompts call the LLM gateway directly.

Designated LangGraph flows:
- **Resume parsing** — conditional self-repair loop (validate → repair ≤ 2 retries → normalize)
- **Match & gap analysis** — parallel fan-out (semantic | skill | experience scorers) → reduce → gap analysis
- **Interview session** — cyclic state machine with Postgres checkpointing (generate → await → evaluate → conditional follow-up → summarize)
- **Roadmap generation** — sequenced RAG pipeline (gaps → prioritize → retrieve → sequence → estimate → validate)

Everything else (single JD extraction, single answer grading outside a session, taxonomy lookups) calls the gateway directly.

## Rationale
- LangGraph adds real value for state persistence, conditional edges, parallel fan-out, and checkpointing.
- For single-shot prompts, LangGraph is pure ceremony: it adds graph compilation overhead, state schema definition, and node boilerplate with zero benefit.
- "LangGraph everywhere" makes simple operations complex and hides the actual prompt engineering behind framework abstractions.

## Rejected Alternatives
- LangGraph everywhere: ceremony without benefit for single-shot calls.
- No framework (hand-rolled): re-implementing checkpointing, branching, and retry logic per flow.

## Consequences
Developers must evaluate each new flow: does it require state, branching, or parallel execution? If yes, make it a graph. If no, call the gateway directly. This decision is documented per flow in `docs/ADRs/`.

## Revisit Trigger
A new flow becomes multi-step or stateful → promote it to a graph.
