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
- subjects: workspace, semester, code, title, current subject-profile version pointer and ordering.
- subject_profile_versions: workspace, subject, immutable syllabus/terminology/output settings and version metadata.
- documents: workspace, subject, category (question_source/reference), original object key, original filename, detected media type, size, SHA-256, status and upload metadata.
- document_processing_runs: workspace, document, parser/OCR versions, configuration, status, error category and timestamps.
- document_originals: optional explicit immutable original-object metadata/revisions if re-upload/versioning is supported; otherwise immutable original fields remain on documents.
- document_pages: workspace, document/run, page or slide index, dimensions and source render key.
- document_blocks: workspace, page, type, order, text/raw payload, coordinates, extraction method and confidence.
- question_groups: workspace, parent/child grouping and source document.
- questions: workspace, stable identity, source block references, question number, hierarchy, marks, ordering and extraction status.
- question_transcriptions: workspace, question, immutable raw transcription reference and separately versioned user-verified text.
- index_versions: workspace, document, processing run, chunker version, embedding model/version, status and timestamps.
- reference_chunks: workspace, index version, source block range, normalized content, unit/section metadata and retrieval fields.
- embeddings: workspace, chunk, model ID/version, dimensions and vector; re-embedding creates a new index/embedding version rather than overwriting provenance.
- answers: workspace, question, selected answer mode and current approved version pointer.
- answer_versions: workspace, immutable generated/edited versions, status, content, model metadata, prompt version and question-transcription version.
- answer_sources: workspace, answer version, reference chunk/block, source location and evidence order.
- processing_jobs: workspace, job type, resource IDs, idempotency key, status, attempts, lease token/expiry and safe error category.
- job_attempts: workspace, job, attempt number, worker identity, start/end, lease outcome and safe failure category.
- exports: workspace, subject/filter, status, output object key, expiry and generation metadata.
- export_items: workspace, export, ordered question ID, exact transcription revision ID and exact answer version ID (nullable for questions-only/answers-only modes).
- usage_events: workspace, operation, model/provider, token/page counts, duration and estimated cost.
- audit_events: workspace where applicable, actor/action/resource metadata without copying sensitive document content.

## 3. Important constraints

- Unique workspace-scoped idempotency keys for upload finalization and job creation.
- Unique original hash per workspace may support deduplication, but storage deletion requires reference counting and race-safe authorization checks.
- Questions have stable IDs independent of transcription versions.
- Raw extraction is immutable and scoped to a processing run. Verified text is a separate append-only revision.
- Answer versions are append-only; edits and regenerations create new versions.
- Subject-profile settings are versioned; generation records the exact profile version used.
- Reference index versions are immutable; a rebuild creates a new version and never retargets existing answer citations.
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
