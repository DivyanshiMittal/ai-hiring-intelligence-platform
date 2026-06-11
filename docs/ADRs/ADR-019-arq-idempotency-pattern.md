# ADR-019 — ARQ Task Handler Idempotency Pattern
 
| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-4, TDD NFR-3 |
 
## Context
ARQ uses at-least-once delivery semantics — a task may be retried after a worker crash. Without a mandatory idempotency pattern, developers implementing new tasks handle this inconsistently, leading to double-processing in production.
 
## Decision
Every ARQ task handler MUST follow the **"gate-then-work" pattern**:
 
### Step 1: Atomic Gate (QUEUED → RUNNING)
```python
async def my_task(ctx: dict, job_id: str) -> None:
    async with get_db_session() as session:
        result = await session.execute(
            select(ProcessingJob)
            .where(
                ProcessingJob.id == job_id,
                ProcessingJob.status == JobStatus.QUEUED,
            )
            .with_for_update(skip_locked=True)
        )
        job = result.scalar_one_or_none()
 
        if job is None:
            return  # Already running or completed by another worker
 
        job.status = JobStatus.RUNNING
        job.started_at = utcnow()
        await session.commit()
        job_entity_id = job.entity_id
 
    # Step 2: Do the work (outside any open session)
    try:
        result = await _do_work(job_entity_id)
        await _mark_succeeded(job_id, result)
    except Exception as exc:
        await _mark_failed(job_id, str(exc))
        raise  # Re-raise so ARQ records the failure and retries per its policy
```
 
### Step 2: Work Phase
The actual work occurs outside any transaction context. Long-running LLM calls must not hold a DB connection.
 
### Step 3: Terminal Transitions
- **Success:** `status = SUCCEEDED`, `result_ref` set, `updated_at` refreshed.
- **Failure:** `status = FAILED`, `error` set, `updated_at` refreshed.
- Both transitions are wrapped in a retry loop (3 attempts, 100 ms backoff) in case the DB is briefly unavailable at write time.
### Stale RUNNING Recovery
A background sweep task runs every 5 minutes:
```python
async def sweep_stale_running_jobs(ctx: dict) -> None:
    """Reset RUNNING jobs that have been running longer than the timeout."""
    timeout = timedelta(minutes=10)
    cutoff = utcnow() - timeout
    async with get_db_session() as session:
        await session.execute(
            update(ProcessingJob)
            .where(
                ProcessingJob.status == JobStatus.RUNNING,
                ProcessingJob.started_at < cutoff,
            )
            .values(status=JobStatus.QUEUED, error="Reset by stale sweep")
        )
        await session.commit()
```
The sweep re-queues the `processing_jobs` row to `QUEUED`; a separate process must re-enqueue the ARQ task (via an admin endpoint or a companion Redis scan).
 
## Consequences
- All task handlers are uniformly idempotent. Code review should reject any handler that does not begin with the gate pattern.
- `SELECT FOR UPDATE SKIP LOCKED` requires the database to support advisory locks (Postgres does). This cannot be ported to MySQL without modification.
- The stale sweep must be registered as a cron task in ARQ's `WorkerSettings.cron_jobs`.
## Revisit Trigger
If the platform adopts a transactional outbox with exactly-once delivery semantics, the gate pattern can be simplified. The gate remains valuable as defence-in-depth even with exactly-once delivery.
 