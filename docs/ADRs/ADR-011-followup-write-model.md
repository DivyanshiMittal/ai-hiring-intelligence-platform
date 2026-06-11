# ADR-011 — Follow-up Questions Inserted On-the-Fly During Session

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-10, architecture.md DD-8 |

## Context
The interview session graph generates adaptive follow-up questions when an answer is weak on a target competency. The `interview_questions` table has `parent_question_id` (self-FK) for follow-ups. The question was: are follow-ups pre-generated (speculative tree) or inserted on-the-fly during the session?

## Decision
Follow-up questions are **inserted on-the-fly** into `interview_questions` during the `plan_followup` node of the LangGraph session graph.

Write contract:
1. The `plan_followup` node generates the follow-up question text and rubric via the LLM gateway.
2. The node inserts a new `interview_questions` row with:
   - `parent_question_id` = the triggering question's ID
   - `generated_from = 'follow_up'`
   - `seq` = next sequence number in the session
3. The LangGraph checkpoint is written **after** the DB insert (so state and DB are consistent at every checkpoint).
4. The graph state carries `current_question_id` (UUID) and `session_question_count` (int). It does not carry the full question object — that is always fetched from the DB by ID.

## Rationale
- Pre-generated speculative trees would populate the `interview_questions` table with rows that may never be asked, polluting the session history and analytics.
- On-the-fly insertion is the honest model: a follow-up only exists if it was actually triggered.
- The Postgres checkpointer guarantees consistency: if the node crashes after the DB write but before the checkpoint, on retry the node re-checks whether a follow-up for this question already exists (idempotent lookup by `parent_question_id`) before inserting.

## Consequences
- The session history (all `interview_questions` for a session) accurately reflects only questions that were presented or will be presented.
- The follow-up insertion is a synchronous DB write inside the graph node — it must complete before the checkpoint. This adds one DB round-trip to the follow-up path but is acceptable (< 20 ms).
- Idempotency: the `plan_followup` node checks `SELECT 1 FROM interview_questions WHERE parent_question_id = :qid` before inserting. If a row exists (prior attempt), it skips insertion.

## Revisit Trigger
If speculative follow-up pre-generation is needed for analytics ("what follow-ups could have been asked"), add a separate `potential_follow_ups` table rather than modifying this write model.
