# ADR-016 — Qdrant Tenant Isolation via TenantScopedQdrantRepository

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Critical Issue C-6 resolution, architecture.md SEC-2 |

## Context
The architecture required server-injected `user_id` filters on all Qdrant queries ("never client-supplied — the vector-store equivalent of row-level security"). Without a structural enforcement mechanism, this was a convention. Conventions fail under deadline pressure. A single missing filter exposes all users' private resume and career data to each other.

## Decision
All Qdrant access is centralised behind a single class: **`TenantScopedQdrantRepository`** in `app/retrieval/qdrant_repository.py`.

### Structural Guarantees

1. **Constructor requires `ActorContext`** — the `user_id` is extracted once at construction time. It is stored as a private attribute. No method signature exposes it.

2. **`_tenant_filter()` is private** — it returns a Qdrant `Filter` with a `FieldCondition` on `user_id`. It is called inside every query and mutation method. It cannot be bypassed or omitted by callers.

3. **No direct Qdrant client imports outside `app/retrieval/`** — enforced by `ruff` import boundary rules in `pyproject.toml`. Any `import qdrant_client` outside `app/retrieval/` is a CI lint failure.

4. **Domain code uses typed methods** — callers call `search_resume_chunks(query_vector, top_k)`, `upsert_resume_chunk(chunk)`. The Qdrant filter structure is an implementation detail invisible to the caller.

5. **Mandatory integration test** — `tests/integration/test_qdrant_tenant_isolation.py`:
   - Creates chunks for `user_a`.
   - Queries the repository instantiated with `user_b`'s actor context.
   - Asserts zero results returned.
   - This test runs in CI on every commit.

### Canonical Structure
```python
# app/retrieval/qdrant_repository.py

class TenantScopedQdrantRepository:
    def __init__(
        self,
        client: AsyncQdrantClient,
        actor: ActorContext,
    ) -> None:
        self._client = client
        self._user_id: str = actor.user_id  # extracted once; private

    # ------------------------------------------------------------------ #
    # Private internals — never exposed to callers
    # ------------------------------------------------------------------ #
    def _tenant_filter(self) -> Filter:
        return Filter(
            must=[FieldCondition(key="user_id", match=MatchValue(value=self._user_id))]
        )

    def _with_tenant(self, caller_filter: Filter | None) -> Filter:
        """Merge an optional caller filter with the mandatory tenant filter."""
        if caller_filter is None:
            return self._tenant_filter()
        return Filter(must=[self._tenant_filter(), caller_filter])

    # ------------------------------------------------------------------ #
    # Public domain-typed interface
    # ------------------------------------------------------------------ #
    async def search_resume_chunks(
        self,
        query_vector: list[float],
        top_k: int,
        additional_filter: Filter | None = None,
    ) -> list[ScoredPoint]:
        return await self._client.search(
            collection_name=CollectionAlias.RESUME_CHUNKS,
            query_vector=query_vector,
            query_filter=self._with_tenant(additional_filter),
            limit=top_k,
        )

    async def upsert_resume_chunk(self, chunk: ResumeChunk) -> None:
        point = PointStruct(
            id=chunk.id,
            vector=chunk.embedding,
            payload={**chunk.payload, "user_id": self._user_id},  # always stamped
        )
        await self._client.upsert(
            collection_name=CollectionAlias.RESUME_CHUNKS,
            points=[point],
        )
```

### Collections Covered
All user-scoped collections use `TenantScopedQdrantRepository`:
- `resume_chunks`
- `jd_chunks`

Shared/reference collections (`skills_taxonomy`, `learning_resources`, `interview_bank`) are **not user-scoped** — they use a separate `SharedQdrantRepository` with no tenant filter and read-only access from domain code.

## Consequences
- Dependency injection: `TenantScopedQdrantRepository` is created per-request in the FastAPI dependency provider, taking the request's `ActorContext`.
- Adding a new user-scoped Qdrant collection requires adding methods to `TenantScopedQdrantRepository` — the filter is automatically included.
- The lint rule makes it impossible to add "just a quick direct Qdrant call" without CI failure.

## Revisit Trigger
If Qdrant introduces native multi-tenancy with server-side enforcement (akin to Postgres RLS), evaluate whether the repository pattern can be simplified while preserving the structural guarantee.
