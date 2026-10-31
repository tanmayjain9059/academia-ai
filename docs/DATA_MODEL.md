# Data model

Status: logical schema proposal. Final SQL migrations follow implementation authorization and database selection.

## 1. Ownership model

Use `workspace_id` as the single canonical tenant key. A workspace is the tenant boundary, including when the first release creates one personal workspace per account. Do not maintain separate mutable `tenant_id` and `workspace_id` ownership columns.

Every workspace-owned row includes `workspace_id`. Enforce membership in the application, tenant-scoped relational constraints and PostgreSQL RLS as defense in depth. RLS must be tested with the actual least-privilege runtime role and pooled connections.

## 2. Core entities

- users: identity-provider subject, email metadata, account status and timestamps.
- workspaces: tenant boundary, owner, plan/limits and settings.
- workspace_members: user, workspace, role and membership state (collaboration can be disabled in MVP).
- semesters: workspace, label, academic year and ordering.
- subjects: workspace, semester, code, title, syllabus profile and ordering.
- documents: workspace, subject, category (question_source/reference), original object key, original filename, detected media type, size, SHA-256, status and upload metadata.
- document_processing_runs: workspace, document, parser/OCR versions, configuration, status, error category and timestamps.
- document_pages: workspace, document/run, page or slide index, dimensions and source render key.
- document_blocks: workspace, page, type, order, text/raw payload, coordinates, extraction method and confidence.
- question_groups: workspace, parent/child grouping and source document.
- questions: workspace, stable identity, source block references, question number, hierarchy, marks, ordering and extraction status.
- question_transcriptions: workspace, question, immutable raw transcription reference and separately versioned user-verified text.
- reference_chunks: workspace, source block range, normalized content, unit/section metadata and retrieval fields.
- embeddings: workspace, chunk, model ID/version, dimensions and vector; re-embedding creates a new version.
- answers: workspace, question, selected answer mode and current approved version pointer.
- answer_versions: workspace, immutable generated/edited versions, status, content, model metadata, prompt version and question-transcription version.
- answer_sources: workspace, answer version, reference chunk/block, source location and evidence order.
- processing_jobs: workspace, job type, resource IDs, idempotency key, status, attempts, lease and error category.
- exports: workspace, subject/filter, status, output object key, expiry and generation metadata; retain exact question/answer version IDs included.
- usage_events: workspace, operation, model/provider, token/page counts, duration and estimated cost.
- audit_events: workspace where applicable, actor/action/resource metadata without copying sensitive document content.

## 3. Important constraints

- Unique workspace-scoped idempotency keys for upload finalization and job creation.
- Unique original hash per workspace may support deduplication, but storage deletion requires reference counting and race-safe authorization checks.
- Questions have stable IDs independent of transcription versions.
- Raw extraction is immutable and scoped to a processing run. Verified text is a separate append-only revision.
- Answer versions are append-only; edits and regenerations create new versions.
- Answer sources reference immutable document processing versions and block IDs.
- Exports reference the exact question and answer versions.
- All foreign-key relationships that cross tenant-owned entities must enforce matching workspace ownership, preferably through composite unique keys and composite foreign keys such as (workspace_id, parent_id).
- Index foreign-key columns used in joins, ownership checks and deletion workflows.
- Deleting a document must not silently orphan answer citations. Use tombstone, purge and retention semantics defined in DATA_INTEGRITY_AND_EVALUATION.md.
- Store timestamps in UTC; render in the user's locale.

## 4. Vector storage

Start with pgvector in the primary PostgreSQL database if the chosen managed Postgres plan supports the required extension and index configuration. Keep embedding rows keyed by workspace, chunk ID, model ID/version and dimensions. Retrieval always filters by workspace and subject before ranking.

Do not introduce a separate vector database until data volume, index performance, operational limits or retrieval isolation needs demonstrate a measurable benefit.

## 5. Sensitive data

Do not collect Aadhaar, PAN, government IDs, health data or unrelated personal information for this academic product. Avoid storing student data that is not necessary for its stated purpose. Document retention, export and deletion policies before public launch.

See [Data integrity, tenant isolation and evaluation specification](DATA_INTEGRITY_AND_EVALUATION.md) for lifecycle, RLS, isolation test and benchmark requirements.
