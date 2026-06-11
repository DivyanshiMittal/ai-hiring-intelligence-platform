# Runbook: Stale Processing Jobs Recovery
 
| | |
|---|---|
| **Project** | Hiring-Copilot |
| **Document** | Runbook — Stale Jobs Recovery |
| **Version** | 1.0 |
| **Phase Deliverable** | Phase 3 |
| **Owner** | Platform Engineering |
| **Related ADRs** | ADR-017, ADR-019 |
 
> **Purpose.** A `processing_jobs` row stuck in `RUNNING` status indefinitely means a worker crashed after acquiring the job but before completing it. The ARQ job in Redis may or may not have been re-queued (depends on whether the crash happened before or after ARQ's heartbeat). This runbook defines the automated sweep, how to diagnose stuck jobs, and how to manually recover them.
 
---
 
## Background
 
Under the gate-then-work pattern (ADR-019), a worker transitions a job from `QUEUED → RUNNING` before beginning work. If the worker process dies after this transition:
 
- The `processing_jobs` row is stuck in `RUNNING`.
- ARQ may re-queue the Redis job (if the worker's heartbeat expired), but the new worker will skip the job because `status != QUEUED`.
- The entity being processed (e.g., a resume) remains in `PENDING` or partially updated state.
- The user's SSE subscription will wait indefinitely for a completion event that never comes.
The **stale job sweep** (ADR-019) is the automated recovery mechanism. This runbook documents it and provides manual recovery procedures for edge cases.
 
---
 
## 1. Automated Stale Sweep
 
The sweep runs as an ARQ cron task every 5 minutes. It resets any `RUNNING` job whose `started_at` is older than 10 minutes back to `QUEUED`.
 
```python
# app/workers/tasks/sweep_stale_jobs.py
 
STALE_THRESHOLD = timedelta(minutes=10)
 
async def sweep_stale_running_jobs(ctx: dict) -> None:
    """
    Cron: resets RUNNING jobs that have exceeded the stale threshold.
    Registered in WorkerSettings.cron_jobs to run every 5 minutes.
    """
    cutoff = utcnow() - STALE_THRESHOLD
    async with get_db_session() as session:
        result = await session.execute(
            update(ProcessingJob)
            .where(
                ProcessingJob.status == JobStatus.RUNNING,
                ProcessingJob.started_at < cutoff,
                ProcessingJob.job_type != JobType.DELETE_USER,  # never auto-retry deletions
            )
            .values(
                status=JobStatus.QUEUED,
                error="Reset by stale sweep — worker likely crashed",
                updated_at=utcnow(),
            )
            .returning(ProcessingJob.id, ProcessingJob.job_type, ProcessingJob.entity_id)
        )
        reset_jobs = result.fetchall()
        await session.commit()
 
    for job in reset_jobs:
        logger.warning(
            "stale_job_reset",
            job_id=str(job.id),
            job_type=job.job_type,
            entity_id=str(job.entity_id),
        )
        # Re-enqueue the ARQ task so a worker picks it up.
        await _reenqueue_job(job)
```
 
The sweep only resets the `processing_jobs` row status. It also re-enqueues the ARQ task so a worker actually picks it up. Without re-enqueueing, the row would sit at `QUEUED` with no worker to process it.
 
---
 
## 2. Diagnosing Stuck Jobs
 
### Check for stuck jobs in the database
 
```sql
-- Jobs stuck in RUNNING for more than 10 minutes.
SELECT
    id,
    user_id,
    job_type,
    entity_id,
    status,
    started_at,
    NOW() - started_at AS stuck_duration,
    error
FROM processing_jobs
WHERE
    status = 'RUNNING'
    AND started_at < NOW() - INTERVAL '10 minutes'
ORDER BY started_at ASC;
```
 
### Check for jobs in a terminal-but-incorrect state
 
```sql
-- Entities still PENDING after their job succeeded (write order bug).
SELECT r.id, r.user_id, r.created_at, pj.status, pj.updated_at
FROM resumes r
LEFT JOIN processing_jobs pj
    ON pj.entity_id = r.id AND pj.job_type = 'parse_resume'
WHERE
    r.embedding_status = 'PENDING'
    AND (pj.status = 'SUCCEEDED' OR pj.id IS NULL)
ORDER BY r.created_at DESC
LIMIT 20;
```
 
### Check ARQ Redis queue depth
 
```bash
# Connect to Redis and check the arq:jobs queue.
redis-cli -u $REDIS_URL LLEN arq:queue:default
# If this is > 0 but workers are idle, workers may be misconfigured.
 
# List all jobs currently in the queue.
redis-cli -u $REDIS_URL LRANGE arq:queue:default 0 -1
```
 
### Check worker health
 
```bash
# Workers should log a heartbeat every 30 seconds.
docker compose logs worker --since 2m | grep '"event":"worker.heartbeat"'
 
# If no heartbeats: worker is down. Check:
docker compose ps worker
docker compose logs worker --tail 50
```
 
---
 
## 3. Manual Recovery Procedures
 
### Recover a single stuck job
 
```bash
# Reset a specific job to QUEUED and re-enqueue it.
python -m app.workers.ops reset-job --job-id <job_uuid>
# This sets status=QUEUED, clears started_at, and enqueues the ARQ task.
# Safe to run even if the worker is still running — the gate (ADR-019)
# prevents double-processing via SELECT FOR UPDATE SKIP LOCKED.
```
 
### Recover all stuck jobs (manual sweep)
 
```bash
# Run the stale sweep immediately (outside of the cron schedule).
python -m app.workers.ops sweep-stale --threshold-minutes 5
# Use a shorter threshold than production (10 min) for emergency recovery.
```
 
### Force-fail a job that cannot be recovered
 
```bash
# If a job keeps crashing and should be marked permanently failed.
python -m app.workers.ops fail-job \
  --job-id <job_uuid> \
  --reason "Manual: unrecoverable error — <description>"
# Sets status=FAILED, clears started_at.
# The entity remains in PENDING; the user will see an error in the UI.
```
 
### Reset all jobs for a specific user (e.g., after a bad deploy)
 
```bash
python -m app.workers.ops reset-user-jobs \
  --user-id <user_uuid> \
  --job-type parse_resume  # optional: filter by type
# Resets RUNNING → QUEUED and re-enqueues all matching jobs.
```
 
---
 
## 4. Job-Type Specific Recovery Notes
 
| Job Type | Entity | Recovery Notes |
|---|---|---|
| `parse_resume` | `resumes` row | Reset to QUEUED; worker re-parses the raw file from MinIO. Idempotent: overwrites `parsed_json`. |
| `parse_jd` | `job_descriptions` row | Same as parse_resume. |
| `score_match` | `match_results` row | Reset to QUEUED; worker re-scores. `model_version` UNIQUE constraint prevents duplicate rows. |
| `generate_roadmap` | `learning_roadmaps` row | Reset to QUEUED; worker regenerates. Existing `roadmap_items` rows are deleted before regeneration. |
| `embed_document` | `resumes` or `job_descriptions` | Reset to QUEUED; embedding is idempotent (same content hash = same vector). |
| `delete_user` | `users` row | **Do NOT auto-reset.** Manual intervention required. Check how far the cascade progressed before failing; complete manually per user-deletion runbook. |
| `reindex_user` | None (bulk op) | Reset to QUEUED; reindex job is fully idempotent. |
 
---
 
## 5. Alerts and SLOs
 
The following conditions should trigger alerts in production:
 
| Condition | Severity | Alert |
|---|---|---|
| Any `RUNNING` job older than 15 minutes (sweep didn't recover it) | Warning | Slack #platform-alerts |
| Same job_id reset by sweep 3+ times | Critical | PagerDuty — indicates a poison-pill job |
| Queue depth > 100 with workers idle | Critical | PagerDuty — worker not consuming |
| `FAILED` job rate > 5% in a 10-minute window | Warning | Slack #platform-alerts |
 
The sweep itself logs `stale_job_reset` events (structlog JSON). Alerting queries against these log events in the observability stack.
 
---
 
## 6. Sweep Configuration Reference
 
| Parameter | Default | Location |
|---|---|---|
| Sweep interval | 5 minutes | `WorkerSettings.cron_jobs` |
| Stale threshold | 10 minutes | `STALE_JOB_THRESHOLD_MINUTES` env var |
| Job types excluded from auto-reset | `delete_user` | `sweep_stale_jobs.py` constant |
| Max requeue attempts before auto-fail | 5 | `MAX_JOB_ATTEMPTS` env var |
 
---
 
*This runbook is a Phase 3 deliverable. The sweep task must be running and tested before Phase 3 is marked complete. Verify by: (1) manually setting a job to RUNNING with a past started_at, (2) waiting for the sweep cron to fire, (3) confirming the job transitions back to QUEUED and is re-enqueued.*
 
 