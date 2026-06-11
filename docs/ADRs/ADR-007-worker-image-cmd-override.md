# ADR-007 — Worker Service Uses API Image with CMD Override

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-1 resolution |
| **Supersedes** | TDD DEP-1 "three images" statement |

## Context
The architecture listed three Docker images (`web`, `api`, `worker`) as Phase 1 deliverables. Worker code lives inside `apps/api/app/workers/`, sharing all domain models, database connections, and LLM gateway code with the API. A separate `Dockerfile.worker` would build a second image from the same source, creating a dependency synchronisation burden.

## Decision
There is **no `Dockerfile.worker`**. The worker service in `docker-compose.yml` uses the `hiring-copilot-api` image with a CMD override:

```yaml
services:
  api:
    build:
      context: ../../apps/api
      dockerfile: ../../infra/docker/Dockerfile.api
    image: hiring-copilot-api
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000

  worker:
    image: hiring-copilot-api     # same image; built once by the api service
    depends_on:
      api:
        condition: service_started
    command: arq app.workers.main.WorkerSettings
```

CI builds one image (`hiring-copilot-api`), tags it, and pushes it. The deploy pipeline runs both containers from the same tag. **TDD DEP-1 is updated** to read: "Two application images: `web` and `api`. The `api` image is reused for the `worker` service via `command` override."

## Rationale
- **Zero dependency drift.** The worker and API always use identical Python packages, domain models, and LLM gateway code. There is no mechanism by which they can diverge.
- **Single CI build step.** One image build, one test run, one vulnerability scan.
- **Simpler rollback.** Rolling back the API image automatically rolls back the worker.
- Worker-specific configuration (concurrency, queue names, retry limits) is supplied via environment variables at runtime, not baked into the image.

## Rejected Alternative
Separate `Dockerfile.worker`: builds from the same source but installs in a separate layer sequence. Packages can diverge if `pyproject.toml` is edited without rebuilding both images in CI. No benefit over the CMD-override approach.

## Consequences
- The `Dockerfile.api` must not bake in a hardcoded `CMD` that is impossible to override. It uses `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]`; the worker Compose service overrides this with `command: arq app.workers.main.WorkerSettings`.
- Any dependency needed only by the worker (e.g., a heavy ML library) is installed in the shared image. If a worker-only dependency is very large, revisit this decision.

## Revisit Trigger
A worker-only dependency is so large that it materially inflates the API container size and affects cold-start time (e.g., adding a local embedding model to the worker). At that point, evaluate a separate worker image with a shared base.
