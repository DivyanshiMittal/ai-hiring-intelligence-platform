# Runbook: User Deletion Cascade
 
| | |
|---|---|
| **Project** | Hiring-Copilot |
| **Document** | Runbook — User Right-to-Deletion |
| **Version** | 1.0 |
| **Phase Deliverable** | Phase 5 |
| **Owner** | Platform Engineering |
| **Compliance** | GDPR Article 17, SEC-3, NFR-5, SEC-7 |
 
> **Purpose.** When a user exercises their right to deletion, all data associated with that user must be purged across PostgreSQL, Qdrant, and MinIO. This runbook defines the exact cascade order, the async job mechanism, and verification steps. The cascade order is determined by foreign-key constraints — leaf tables first, root last.
 
---
 
## Trigger Sources
 
A deletion may be initiated via:
1. **User self-service** — user clicks "Delete my account" in the dashboard (POST `/account/delete`).
2. **Admin action** — admin issues a deletion via the admin API.
3. **Retention policy** — an automated sweep deletes accounts inactive beyond the configured retention window (future, Phase 5+).
All three paths produce the same artifact: a `processing_jobs` row of type `delete_user` following the write-order invariant (ADR-017).
 
---
 
## 1. Initiating a Deletion
 
```python
# Service layer — app/account/service.py
async def request_user_deletion(actor: ActorContext, session: AsyncSession) -> str:
    """
    Creates a deletion job following ADR-017 write-order invariant.
    Returns job_id for SSE tracking.
    """
    job_id = new_id()
    async with session.begin():
        session.add(ProcessingJob(
            id=job_id,
            user_id=actor.user_id,
            job_type=JobType.DELETE_USER,
            entity_id=actor.user_id,   # the entity being deleted is the user itself
            status=JobStatus.QUEUED,
        ))
    # Commit happens on context manager exit — THEN enqueue (ADR-017)
    await arq_redis.enqueue_job("delete_user_task", job_id=job_id)
    return job_id
```
 
The HTTP response returns `202 { job_id }`. The client may poll `/jobs/{job_id}` or subscribe to SSE.
 
---
 
## 2. Cascade Deletion Order
 
The order is leaf-to-root following all foreign-key relationships. Violating this order causes FK constraint errors.
 
```
Phase A — LLM and analytics logs (no FK dependencies on other user tables)
  1.  llm_audit_log          WHERE user_id = :uid
  2.  analytics_events        WHERE user_id = :uid
 
Phase B — Evaluation and interview leaf rows
  3.  answer_evaluations      WHERE question_id IN
                                (SELECT id FROM interview_questions
                                 WHERE session_id IN
                                   (SELECT id FROM interview_sessions WHERE user_id = :uid))
  4.  interview_questions     WHERE session_id IN
                                (SELECT id FROM interview_sessions WHERE user_id = :uid)
  5.  interview_sessions      WHERE user_id = :uid
 
Phase C — Roadmap leaf rows
  6.  roadmap_items           WHERE roadmap_id IN
                                (SELECT id FROM learning_roadmaps WHERE user_id = :uid)
  7.  learning_roadmaps       WHERE user_id = :uid
 
Phase D — Match and gap leaf rows
  8.  skill_gaps              WHERE match_id IN
                                (SELECT id FROM match_results WHERE user_id = :uid)
  9.  match_results           WHERE user_id = :uid
 
Phase E — Resume and JD leaf rows
  10. resume_skills           WHERE resume_id IN
                                (SELECT id FROM resumes WHERE user_id = :uid)
  11. jd_requirements         WHERE jd_id IN
                                (SELECT id FROM job_descriptions WHERE user_id = :uid)
  12. resumes                 WHERE user_id = :uid
  13. job_descriptions        WHERE user_id = :uid
 
Phase F — Document registry
  14. documents               WHERE user_id = :uid
      (also collects storage_key values for MinIO deletion in Phase H)
 
Phase G — Jobs (delete after entity rows so SSE tracking remains valid during the job)
  15. processing_jobs         WHERE user_id = :uid
                              AND id != :current_job_id  -- keep the deletion job itself alive
 
Phase H — External stores (async, after Postgres commit)
  16. Qdrant points           user_id filter on resume_chunks, jd_chunks
  17. MinIO objects           all storage_key values collected in Phase F
 
Phase I — Root row (last)
  18. users                   WHERE id = :uid
 
Phase J — Self-deletion (after all other work is confirmed complete)
  19. processing_jobs         WHERE id = :current_job_id  -- delete the deletion job itself
```
 
---
 
## 3. Worker Implementation Pattern
 
```python
# app/workers/tasks/delete_user.py
 
async def delete_user_task(ctx: dict, job_id: str, _trace_context: dict | None = None) -> None:
    """
    Cascading user deletion following the order in docs/runbooks/user-deletion.md.
    Implements ADR-019 gate-then-work pattern.
    Implements ADR-020 trace propagation.
    """
    # --- Gate (ADR-019) ---
    parent_ctx = propagate.extract(_trace_context or {})
    with tracer.start_as_current_span("worker.delete_user", context=parent_ctx):
 
        async with get_db_session() as session:
            job = await _acquire_job(session, job_id)
            if job is None:
                return
            user_id = str(job.entity_id)
 
        # --- Work phases (outside transaction for long-running operations) ---
 
        # Phases A–G: Postgres deletions in FK-safe order
        storage_keys = await _delete_postgres_data(user_id, job_id)
 
        # Phase H: External stores (after Postgres commit, so Postgres is truth)
        await _delete_qdrant_points(user_id)
        await _delete_minio_objects(storage_keys)
 
        # Phase I: Root user row
        async with get_db_session() as session:
            await session.execute(delete(User).where(User.id == user_id))
            await session.commit()
 
        # Phase J: Self-deletion of the job row
        async with get_db_session() as session:
            await session.execute(
                delete(ProcessingJob).where(ProcessingJob.id == job_id)
            )
            await session.commit()
```
 
---
 
## 4. Idempotency
 
The deletion task is idempotent. If it is interrupted and retried:
 
- **Postgres deletes** use `DELETE WHERE` — already-deleted rows produce zero-row deletes, not errors.
- **Qdrant deletes** use `delete_vectors` with the `user_id` filter — deleting non-existent points is a no-op.
- **MinIO deletes** use `remove_object` — deleting a non-existent object returns a 404 that is caught and logged, not raised.
- The gate (ADR-019) prevents two workers running the deletion simultaneously.
---
 
## 5. Manual Deletion (Emergency / GDPR Request)
 
If the async job fails repeatedly and manual intervention is required:
 
```bash
# Connect to the Postgres instance.
psql $DATABASE_URL
 
-- Run the cascade in the exact order from §2.
-- Replace :uid with the target user's UUID.
BEGIN;
 
DELETE FROM llm_audit_log         WHERE user_id = :'uid';
DELETE FROM analytics_events      WHERE user_id = :'uid';
-- ... (follow §2 order exactly) ...
DELETE FROM users                  WHERE id = :'uid';
 
COMMIT;
```
 
```bash
# Delete Qdrant points for the user.
python -m app.retrieval.ops delete-user-points --user-id <uid>
 
# List and delete MinIO objects for the user.
# Objects are namespaced: resumes/<user_id>/<document_id>
mc ls minio/hiring-copilot/resumes/<uid>/
mc rm --recursive --force minio/hiring-copilot/resumes/<uid>/
mc rm --recursive --force minio/hiring-copilot/jd/<uid>/
```
 
---
 
## 6. Verification
 
```bash
# Confirm no Postgres rows remain for the user.
python -m app.account.ops verify-deletion --user-id <uid>
# Checks all 18 tables in §2. Reports any remaining rows as FAIL.
 
# Confirm no Qdrant points remain.
python -m app.retrieval.ops count-user-points --user-id <uid>
# Expected: 0 across all collections.
 
# Confirm no MinIO objects remain.
mc ls minio/hiring-copilot/resumes/<uid>/  # should return empty
mc ls minio/hiring-copilot/jd/<uid>/       # should return empty
```
 
---
 
## 7. Audit Trail
 
Even after deletion, the following records are retained for compliance:
 
- **`llm_audit_log`** rows are deleted (they contain `user_id`; their retention would re-identify the deleted user). Cost aggregates derived from them are preserved in materialized snapshots without user_id attribution.
- The deletion event itself is logged to the application log (structlog) with `event=user_deleted, user_id=<uid>` before the `users` row is dropped. This log line is the audit trail; it does not contain PII beyond the UUID.
- If a legal hold is required on a specific account, the deletion must be blocked at the service layer by an admin flag before any deletion job is created. This is a future compliance feature (SEC-7).
---
 
*This runbook is a Phase 5 deliverable. It must be tested end-to-end in a staging environment — including the verification step — before Phase 5 is marked complete.*