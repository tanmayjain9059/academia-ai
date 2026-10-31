# Data model

Status: logical schema proposal. Final SQL migrations follow implementation authorization and database selection.

## 1. Ownership model

Use a workspace/tenant boundary even if the first release creates one personal workspace per account. Every tenant-owned row includes tenant_id. Authorization must be enforced in application queries and, where practical, PostgreSQL row-level security as defense in depth.

## 2. Core entities

- users: identity-provider subject, email metadata, account status and timestamps.
- workspaces: tenant boundary, owner, plan/limits and settings.
- workspace_members: user, workspace, role and membership state (collaboration can be disabled in MVP).
- semesters: workspace, label, academic year and ordering.
- subjects: workspace, semester, code, title, syllabus profile and ordering.
- documents: workspace, subject, category (question_source/reference), original object key, original filename, detected media type, size, SHA-256, status and upload metadata.
- document_processing_runs: document, parser/OCR versions, configuration, status, error category and timestamps.
- document_pages: document/run, page or slide index, dimensions and source render key.
- document_blocks: page, type, order, text/raw payload, coordinates, extraction method and confidence.
- question_groups: parent/child grouping and source document.
- questions: stable identity, source block references, question number, hierarchy, marks, ordering and extraction status.
- question_transcriptions: immutable raw transcription and separately versioned user-verified text.
- reference_chunks: source block range, normalized content, unit/section metadata and retrieval fields.
- answers: question, owner/workspace, selected answer mode and current approved version pointer.
- answer_versions: immutable generated/edited versions, status, content, model metadata and prompt version.
- answer_sources: answer version, reference chunk/block, source location and evidence order.
- processing_jobs: tenant, job type, resource IDs, idempotency key, status, attempts, lease and error category.
- exports: tenant, subject/filter, status, output object key, expiry and generation metadata.
- usage_events: tenant, operation, model/provider, token/page counts, duration and estimated cost.
- audit_events: security-relevant actor/action/resource metadata without copying sensitive document content.

## 3. Important constraints

- Unique workspace-scoped idempotency keys for upload finalization and job creation.
- Unique original hash per workspace may support deduplication, but storage deletion requires reference counting and authorization checks.
- Questions have stable IDs independent of transcription versions.
- Raw extraction is immutable. Verified text is a separate versioned entity.
- Answer versions are append-only; edits and regenerations create new versions.
- Answer sources reference immutable document processing versions and block IDs.
- Exports reference the exact question and answer versions included.
- All foreign-key relationships that cross tenant-owned entities must enforce matching workspace/tenant ownership, using composite keys or equivalent constraints where practical.
- Deleting a document must not silently orphan answer citations. Define soft-delete, retention and purge semantics.
- Store timestamps in UTC; render in the user's locale.

## 4. Vector storage

Start with pgvector in the primary PostgreSQL database if the chosen managed Postgres plan supports the required extension and index configuration. Keep embedding rows keyed by chunk ID, model ID/version and dimensions. Re-embedding creates new vectors rather than mutating provenance.

Do not introduce a separate vector database until data volume, index performance, operational limits or retrieval isolation needs demonstrate a measurable benefit.

## 5. Sensitive data

Do not collect Aadhaar, PAN, government IDs, health data or unrelated personal information for this academic product. Avoid storing student data that is not necessary for its stated purpose. Document retention, export and deletion policies before public launch.
