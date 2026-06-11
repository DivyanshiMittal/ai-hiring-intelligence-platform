# ADR-010 — Skills Taxonomy Seeded from ESCO with MVT Stub for Phase 1

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | Recommended Issue R-3, TDD §12 (open question) |

## Context
`TDD.md §12` flagged the taxonomy seed source as an open question. Phase 1 said "can be stubbed." But Phase 4 requires ≥ 90% skill normalization accuracy, which depends on taxonomy completeness. The gap between "stub" and "production seed" needed a concrete plan.

## Decision
A **two-stage seeding approach**:

**Phase 1 — Minimum Viable Taxonomy (MVT):**
A hand-curated seed of ~500 canonical technology and professional skills, sufficient to pass Phase 4's golden-set tests. Delivered as a data migration (`0002_seed_mvt_taxonomy.py`). Covers: programming languages, web frameworks, databases, cloud platforms, AI/ML tools, common soft skills, and the top 50 job-category domains.

**Phase 4 — Full ESCO Seed:**
The full ESCO Skills Pillar (~13,890 concepts, CC BY 4.0 licence) is loaded via a second data migration (`0003_seed_esco_taxonomy.py`). ESCO records are mapped to the `skills_taxonomy` schema: `canonical_name`, `display_name`, `category`, `aliases[]`. Records that conflict with the MVT stub are merged (ESCO display name + aliases are added; existing canonical_name is preserved).

**Ongoing — Curation Queue:**
Skills encountered during resume parsing that don't match any canonical entry are stored as `raw_mention` with null `skill_canonical`. A curation UI (Phase 4+) surfaces these for admin review. Accepted entries become new `skills_taxonomy` rows.

## Rationale
- Blocking Phase 1 on a full ESCO import adds risk to the foundation phase with no Phase 1 deliverable benefit.
- The MVT covers the skills most likely to appear on tech resumes — sufficient for Phase 4's golden set.
- Using ESCO (CC BY 4.0) avoids licensing risk and provides a well-structured, maintained taxonomy.
- Delivering seeds as Alembic data migrations (not `init.sql` or separate scripts) ensures they run on every environment, are tracked in `alembic_version`, and are not re-run.

## Consequences
- `infra/postgres/seed/` contains the MVT CSV (source of truth for the migration).
- Phase 4 adds the ESCO CSV and migration.
- The `skills_taxonomy` table must be stable from Phase 1; schema changes after Phase 4 seeding require careful migration.

## Revisit Trigger
If a domain-specific taxonomy (e.g., legal, medical) is needed beyond ESCO's coverage, add a domain-specific data migration following the same pattern.
