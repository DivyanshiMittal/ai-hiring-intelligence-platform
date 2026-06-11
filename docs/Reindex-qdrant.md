# Runbook: Reindex Qdrant from PostgreSQL
 
| | |
|---|---|
| **Project** | Hiring-Copilot |
| **Document** | Runbook — Qdrant Full Reindex |
| **Version** | 1.0 |
| **Phase Deliverable** | Phase 5 |
| **Owner** | Platform / ML Engineering |
| **Last Tested** | — (update when run) |
 
> **Purpose.** Qdrant is a derived, rebuildable index (DD-2, ADR-016). This runbook is the operational proof of that claim. It must be executable by any engineer from the commands below, with no hidden steps. Run it after: a Qdrant data loss event, an embedding model upgrade, a collection schema change, or a new environment bootstrap.
 
---
 
## Prerequisites
 
- Access to the Postgres database (read access on `resumes`, `resume_skills`, `job_descriptions`, `skills_taxonomy`, `learning_resources`, `interview_bank`).
- Access to the Qdrant instance (write access to collections).
- The application service account credentials (same ones the API uses).
- The `hiring-copilot-api` Docker image at the target tag.
- Python environment with `uv` or the API's virtualenv activated.
---
 
## When to Run This Runbook
 
| Trigger | Action |
|---|---|
| Qdrant volume wiped or corrupted | Full reindex (all collections) |
| Embedding model version upgraded | New collection + alias swap (§4) |
| Collection schema changed (new payload fields) | Full reindex of affected collection(s) |
| New environment provisioned | Full reindex after migrations run |
| Stale index suspected (query quality degraded) | Reindex affected collection(s) |
 
---
 
## 1. Safety Checks Before Starting
 
```bash
# 1. Confirm the target Qdrant instance and confirm you are NOT pointing at production
#    unless this is an intentional production reindex.
echo $QDRANT_URL
 
# 2. Confirm Postgres is reachable and migrations are current.
alembic current
# Expected: "head" on the target database.
 
# 3. Confirm current Qdrant collection aliases (to know what is live).
python -m app.retrieval.ops list-aliases
# Example output:
#   resume_chunks       → resume_chunks_text-embedding-004
#   jd_chunks           → jd_chunks_text-embedding-004
#   skills_taxonomy     → skills_taxonomy_text-embedding-004
#   learning_resources  → learning_resources_text-embedding-004
#   interview_bank      → interview_bank_text-embedding-004
```
 
---
 
## 2. Full Reindex (All Collections)
 
Use this when recovering from data loss or bootstrapping a new environment.
 
```bash
# Step 1: Create new versioned collections.
#   The naming convention is {collection_name}_{embedding_model_version}.
#   The current embedding model version is in settings.llm.embedding_model_version.
python -m app.retrieval.ops create-collections --version $(python -m app.core.config print-embedding-version)
# Creates: resume_chunks_<ver>, jd_chunks_<ver>, skills_taxonomy_<ver>,
#          learning_resources_<ver>, interview_bank_<ver>
# Does NOT affect the existing collections or aliases.
 
# Step 2: Seed reference collections (skills taxonomy, learning resources, interview bank).
#   These are not user-scoped. They are loaded once from static data.
python -m app.retrieval.ops seed-reference-collections --version <ver>
# Reads from: skills_taxonomy, learning_resources, interview_bank Postgres tables.
# Embeds and upserts into new Qdrant collections.
# Estimated time: ~5-15 minutes depending on collection size.
 
# Step 3: Reindex user-scoped collections (resume_chunks, jd_chunks).
#   Reads all rows from Postgres; batches embedding calls; upserts to Qdrant.
python -m app.retrieval.ops reindex-user-data --collection resume_chunks --version <ver>
python -m app.retrieval.ops reindex-user-data --collection jd_chunks --version <ver>
# Each command processes all rows for all users.
# Progress is logged to stdout (structlog JSON).
# Estimated time: ~1 min per 1000 resume chunks at batch size 100.
# Safe to interrupt and resume: uses cursor pagination; already-upserted points
#   are idempotent (same point ID = overwrite).
 
# Step 4: Verify counts match.
python -m app.retrieval.ops verify-counts --version <ver>
# Compares Postgres row counts vs Qdrant point counts per collection.
# Reports any discrepancy. Expected output: all collections PASS.
 
# Step 5: Swap aliases to the new collections.
python -m app.retrieval.ops swap-aliases --version <ver>
# Atomically updates all collection aliases to point to <ver> collections.
# Zero-downtime: the alias swap is a single Qdrant API call per alias.
# The application starts reading from the new collections immediately.
 
# Step 6: (After 24-hour grace period) Delete old collections.
python -m app.retrieval.ops delete-old-collections --keep-versions 1
# Keeps the most recent N versions. Deletes everything older.
# Run manually after confirming the new collections are healthy in production.
```
 
---
 
## 3. Single-Collection Reindex
 
Use this when only one collection needs rebuilding (e.g., a schema change to resume payloads only).
 
```bash
# Reindex only resume_chunks.
python -m app.retrieval.ops reindex-user-data \
  --collection resume_chunks \
  --version <ver> \
  --swap-alias  # automatically swaps alias after completion
```
 
---
 
## 4. Embedding Model Upgrade (Zero-Downtime)
 
This procedure upgrades the embedding model without dropping the existing index. The alias pattern (`{collection_name}` always points to the active collection) is the key mechanism.
 
```bash
# Before starting: update settings.llm.embedding_model_version to the new version.
# This is a config change only; no code change required.
 
NEW_VER="text-embedding-005"  # example new version
 
# Step 1: Create new collections under the new version.
python -m app.retrieval.ops create-collections --version $NEW_VER
 
# Step 2: Seed reference collections with new embeddings.
python -m app.retrieval.ops seed-reference-collections --version $NEW_VER
 
# Step 3: Reindex user data with new embeddings.
python -m app.retrieval.ops reindex-user-data --collection resume_chunks --version $NEW_VER
python -m app.retrieval.ops reindex-user-data --collection jd_chunks --version $NEW_VER
 
# Step 4: Verify counts.
python -m app.retrieval.ops verify-counts --version $NEW_VER
 
# Step 5: Swap aliases.
#   From this point, all new queries use the new embeddings.
#   Old queries in flight complete against the old collection (it still exists).
python -m app.retrieval.ops swap-aliases --version $NEW_VER
 
# Step 6: Deploy the new application version that uses the new embedding model
#         for query-time embedding generation.
#   IMPORTANT: the application must use the same model for query embeddings
#              as was used for index embeddings. Mixing models produces
#              meaningless similarity scores.
#   The embedding_model_version config change from Step 0 ensures this.
 
# Step 7: After 24h grace period, delete old collections.
python -m app.retrieval.ops delete-old-collections --keep-versions 1
```
 
---
 
## 5. Verification Queries
 
After any reindex, confirm retrieval quality with these spot checks:
 
```bash
# Semantic search smoke test: a known resume should retrieve itself.
python -m app.retrieval.ops smoke-test \
  --collection resume_chunks \
  --resume-id <a known resume_id> \
  --expect-top-1-match
 
# Cross-tenant isolation test (mandatory after any reindex).
python -m app.retrieval.ops test-tenant-isolation
# Creates two test users, inserts data for user A, queries as user B.
# Expected: zero results for user B. Any result is a CRITICAL FAILURE — page on-call.
 
# Query latency check.
python -m app.retrieval.ops benchmark --collection resume_chunks --queries 100
# Expected: p95 < 50ms for HNSW search with payload filter.
```
 
---
 
## 6. Rollback
 
If the new collections produce degraded results, swap the alias back to the previous version:
 
```bash
PREV_VER="text-embedding-004"  # the previous version
 
python -m app.retrieval.ops swap-aliases --version $PREV_VER
# Alias is back on the previous collections within seconds.
# No data was lost; both old and new collections exist until delete-old-collections runs.
```
 
---
 
## 7. Estimated Timings (Reference)
 
| Scale | resume_chunks reindex | jd_chunks reindex | Reference collections |
|---|---|---|---|
| 1,000 users, avg 50 chunks/resume | ~5 min | ~2 min | ~3 min (one-time) |
| 10,000 users | ~50 min | ~20 min | Same |
| 100,000 users | ~8 hours | ~3 hours | Same |
 
For large reindexes, run in a screen/tmux session or as a Kubernetes Job with resource limits.
 
---
 
## 8. Troubleshooting
 
| Symptom | Likely Cause | Action |
|---|---|---|
| `verify-counts` shows mismatch | Batch failed mid-run | Re-run `reindex-user-data` for that collection; it is idempotent |
| Query returns other users' data | Tenant filter missing in new code | Halt immediately; do not swap alias; fix `TenantScopedQdrantRepository`; re-run isolation test |
| Very low similarity scores after model upgrade | Query encoder not updated to match index encoder | Ensure `embedding_model_version` config matches in both indexer and query path; re-deploy |
| `smoke-test` top-1 not self-match | Chunking changed; IDs don't match | Expected if chunk IDs changed; verify retrieval quality manually with known queries |
 
---
 
*This runbook is a Phase 5 deliverable. It must be tested end-to-end in a staging environment before Phase 5 is marked complete.*