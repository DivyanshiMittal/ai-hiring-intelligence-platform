# ADR-014 — ClamAV Sidecar for Antivirus Scanning

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-06-10 |
| **Deciders** | Principal AI Architect |
| **Source** | architecture.md SEC-5 ("antivirus scan hook on upload") |

## Context
SEC-5 requires an antivirus scan hook on file upload. The platform accepts arbitrary PDF and DOCX files from untrusted users. Without AV scanning, a malicious file could reach the document extraction pipeline and potentially exploit parsing libraries.

## Decision
ClamAV runs as a **sidecar container** in Docker Compose. The API worker service communicates with ClamAV via the `clamd` TCP socket.

**Phase 1:** ClamAV service is defined in `docker-compose.yml` with the `clamav/clamav` image. The connection details are in the API config. The scan hook in the ingestion service is **scaffolded but not activated** — it logs a warning if `CLAMAV_ENABLED=false` (the Phase 1 default).

**Phase 5 (Hardening):** `CLAMAV_ENABLED=true` is set in all non-development environments. The scan runs asynchronously in the ARQ worker *before* the document is passed to the text extraction pipeline. A scan failure marks the `processing_jobs` row as `FAILED` with `error_code: MALWARE_DETECTED` and deletes the raw file from MinIO.

**Scan placement:** The scan occurs in the ARQ parse worker, not in the HTTP upload handler. This preserves the < 500 ms upload response SLO (FR-1) while still scanning before any parsing is done on the content.

## Consequences
- The Compose file has 8 containers in Phase 1, not 7. ClamAV adds ~200 MB to local dev memory footprint.
- ClamAV's virus database is updated on container start via the `clamav/clamav` image's built-in `freshclam`. In production, a scheduled `freshclam` job must run.
- Scan latency: typically 50–200 ms for a 10 MB file. This is inside the 30 s parse job SLO.

## Revisit Trigger
If a cloud-native AV service (e.g., AWS GuardDuty Malware Protection for S3) is available in the deployment environment, prefer it over the ClamAV sidecar — it scans at the storage layer without application involvement.
