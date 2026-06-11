# ADR-017 — Processing Jobs Write-Order Invariant

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-7 resolution, TDD DD-5, NFR-3 |

## Context
The platform creates `processing_jobs` rows to drive async AI operations, and enqueues ARQ tasks that workers pick up and process. The write ordering between entity rows, `processing_jobs` rows, and the ARQ enqueue call was unspecified. Without a defined order, a race condition exists: a worker can attempt to read or update rows that haven't been committed yet.

## Decision
The **mandatory write-order invariant** for all operations that create an entity and trigger async processing:

```
BEGIN TRANSACTION
  1. INSERT entity row(s)         (e.g., documents, resumes)
  2. INSERT processing_jobs row   (status=QUEUED, entity_id=<entity_id>)
COMMIT TRANSACTION
――――――――――――――――――――――――――――――――――
3. Enqueue ARQ task               ← OUTSIDE the transaction; AFTER commit
```

**The ARQ enqueue call is always outside the transaction and always after commit. This is inviolable.**

### Mandatory Worker Precondition
Every ARQ task handler MUST begin with an atomic status transition before performing any work:

```python
async def parse_resume_task(ctx: dict, job_id: str) -> None:
    async with get_db_session() as session:
        # Atomic transition: QUEUED → RUNNING
        # SELECT FOR UPDATE prevents two workers from picking up the same job
        job = await session.execute(
            select(ProcessingJob)
            .where(ProcessingJob.id == job_id)
            .where(ProcessingJob.status == JobStatus.QUEUED)
            .with_for_update(skip_locked=True)
        )
        job = job.scalar_one_or_none()

        if job is None:
            # Already picked up by another worker, or row doesn't exist.
            # In either case: do nothing. ARQ will not retry (no exception raised).
            return

        job.status = JobStatus.RUNNING
        job.started_at = utcnow()
        await session.commit()

    # --- actual work begins here, outside the transition transaction ---
    await _do_parse(job_id)
```

### Failure Recovery
| Scenario | Outcome |
|---|---|
| DB transaction fails (step 1 or 2) | No ARQ task enqueued. Entity does not exist. No worker runs. Clean state. |
| DB commits but ARQ enqueue fails (step 3) | `processing_jobs` row exists in `QUEUED` status. No worker picks it up. Operator re-enqueues manually by job_id. No data loss. |
| Worker crashes after RUNNING transition but before completion | `processing_jobs` row stuck in `RUNNING`. ARQ job in Redis is re-queued by ARQ's retry mechanism. On retry, `SELECT ... WHERE status = QUEUED` finds no row (status is RUNNING) — task exits immediately. Resolution: a background sweep resets RUNNING jobs older than a timeout (e.g., 10 min) back to QUEUED. |
| Worker completes successfully | `processing_jobs` row transitions to `SUCCEEDED`; entity row updated. |

### Application to All Async Flows
This invariant applies to every flow that creates an entity and enqueues processing:

| Flow | Entity rows created | Job type |
|---|---|---|
| Resume upload | `documents`, `resumes(PENDING)` | `parse_resume` |
| JD upload | `documents`, `job_descriptions(PENDING)` | `parse_jd` |
| Match scoring | `match_results(PENDING)` | `score_match` |
| Roadmap generation | `learning_roadmaps(PENDING)` | `generate_roadmap` |
| User deletion | _(no new entity)_ | `delete_user` |
| Qdrant reindex | _(no new entity)_ | `reindex_user` |

## Consequences
- Service layer code must never call `arq.enqueue()` inside a SQLAlchemy transaction context. Code review and linting should flag `enqueue` calls inside `async with session:` blocks.
- The stale RUNNING jobs sweep is a background maintenance task that must be implemented in Phase 3 alongside the first use of `processing_jobs`.
- ARQ task handlers are idempotent by design: if the same task runs twice (e.g., after the stale sweep re-queues it), the `SELECT FOR UPDATE WHERE status = QUEUED` gate prevents double-processing.

## Revisit Trigger
If the platform adopts a transactional outbox pattern (writing to an `outbox` table inside the transaction and having a separate relay process enqueue from the outbox), this invariant is subsumed by that pattern. The outbox pattern would be adopted if the platform moves to a message broker (Kafka, RabbitMQ) rather than Redis + ARQ.
