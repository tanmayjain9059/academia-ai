# API specification

Status: proposed resource contract. Exact payload schemas and OpenAPI definitions will be authored alongside implementation.

## 1. Conventions

- Prefix: `/api/v1`.
- JSON request/response except binary upload/download handled through private object storage.
- Authentication required except health and explicitly public metadata routes.
- `workspace_id` is the canonical ownership boundary. A path workspace ID requests a scope; it does not prove membership. Membership and role are verified server-side from the authenticated principal.
- Use opaque resource IDs, cursor pagination, request IDs and idempotency keys.
- Use optimistic concurrency (ETag/If-Match or expected version IDs) for operations that update a current pointer or create a revision from an existing version.
- Return stable error codes; do not leak stack traces, signed URLs or private resource existence across workspaces.
- Detailed lifecycle, idempotency, cancellation, deletion and worker-lease rules are specified in [API lifecycle contract](API_LIFECYCLE_CONTRACT.md).

## 2. Planned resource families

### Workspaces, semesters and subjects

- `POST /workspaces`
- `GET /workspaces`
- `GET /workspaces/{workspace_id}`
- `PATCH /workspaces/{workspace_id}`
- `POST /workspaces/{workspace_id}/deletion-requests`
- `POST /workspaces/{workspace_id}/semesters`
- `GET /workspaces/{workspace_id}/semesters`
- `PATCH /semesters/{semester_id}`
- `DELETE /semesters/{semester_id}` (only with explicit dependency handling)
- `POST /subjects`
- `GET /subjects`
- `GET /subjects/{subject_id}`
- `PATCH /subjects/{subject_id}`
- `DELETE /subjects/{subject_id}` (archive or explicit dependency handling)

### Documents and processing

- `POST /documents/upload-intents`
- `POST /documents/{document_id}/finalize`
- `GET /documents`
- `GET /documents/{document_id}`
- `POST /documents/{document_id}/deletion-requests`
- `POST /documents/{document_id}/processing-runs` (reprocess; creates a new immutable run)
- `GET /documents/{document_id}/processing-runs`
- `GET /documents/{document_id}/processing-runs/{run_id}`
- `GET /documents/{document_id}/processing-runs/{run_id}/pages`
- `GET /documents/{document_id}/processing-runs/{run_id}/blocks`

Upload flow: API authorizes and creates a pending record, returns a short-lived signed PUT URL for a private object, client uploads directly, then finalization verifies object existence, actual size, file signature and checksum where available. The API creates a durable processing job/outbox event only after validation.

### Question extraction and transcription review

- `POST /documents/{document_id}/processing-runs/{run_id}/question-extractions`
- `GET /questions` (filter by subject, document, run and extraction status)
- `GET /questions/{question_id}`
- `POST /questions/{question_id}/transcription-revisions`
- `GET /questions/{question_id}/transcription-revisions`
- `POST /questions/{question_id}/transcription-revisions/{revision_id}/verify`
- `POST /questions/{question_id}/transcription-revisions/{revision_id}/select`

Question identity is stable; text is held in append-only transcription revisions. Concurrent edits require expected revision IDs. Verification does not edit the source file or raw extraction.

### Reference indexing and retrieval

- `POST /documents/{document_id}/processing-runs/{run_id}/indexing-jobs`
- `GET /documents/{document_id}/index-versions`
- `GET /index-versions/{index_version_id}`
- `GET /index-versions/{index_version_id}/chunks`

Indexing is asynchronous and versioned. Each chunk/embedding records workspace, subject, document, processing run, source blocks, chunker version and embedding model/version. Retrieval filters workspace and subject before ranking.

### Answers and answer versions

- `POST /answers/generate`
- `GET /questions/{question_id}/answers`
- `GET /answers/{answer_id}`
- `GET /answers/{answer_id}/versions`
- `GET /answer-versions/{version_id}`
- `POST /answers/{answer_id}/versions` (create user-edited version from a parent version)
- `POST /answers/{answer_id}/regenerations`
- `POST /answer-versions/{version_id}/approval`
- `GET /answer-versions/{version_id}/sources`

Generation pins the question transcription revision, selected source/index versions, answer mode, marks/length target and subject-profile version. Each generation/edit creates a new immutable answer version. Approval updates a current pointer with optimistic concurrency; it does not mutate answer content.

### Jobs, exports and account privacy

- `GET /jobs/{job_id}`
- `POST /jobs/{job_id}/cancellation-requests`
- `POST /jobs/{job_id}/retry-requests` (only for eligible terminal failures; creates a traceable attempt)
- `POST /exports`
- `GET /exports`
- `GET /exports/{export_id}`
- `POST /exports/{export_id}/download-requests`
- `POST /exports/{export_id}/deletion-requests`
- `POST /account/data-export-requests`
- `POST /account/deletion-requests`
- `GET /account/deletion-requests/{request_id}`

Download request re-authorizes the caller and returns a short-lived, object-specific URL. Do not store or log signed URLs as durable resource data.

## 3. Job contract

Expose job ID, status, safe progress stage, user-facing message, created/updated timestamps, attempt count and retryability. State transitions are persisted and conditional on expected state and lease token. Queue delivery is at-least-once; workers must be idempotent. Cancellation is cooperative. Never expose raw tracebacks, prompts, private document text or provider credentials.

## 4. Error contract

Use a consistent envelope with `code`, `message`, `request_id` and optional `field_errors`. Expected codes include:
- `AUTH_REQUIRED`
- `FORBIDDEN`
- `NOT_FOUND`
- `VALIDATION_ERROR`
- `UNSUPPORTED_FILE`
- `FILE_TOO_LARGE`
- `FILE_SIGNATURE_MISMATCH`
- `QUOTA_EXCEEDED`
- `IDEMPOTENCY_CONFLICT`
- `VERSION_CONFLICT`
- `JOB_CONFLICT`
- `RATE_LIMITED`
- `MODEL_UNAVAILABLE`
- `PROCESSING_FAILED`
- `EXPORT_FAILED`
- `RESOURCE_DELETION_PENDING`

Do not reveal whether a private resource exists when the caller is not authorized to access it.

## 5. API security requirements

- Validate request bodies, query parameters, pagination bounds and allowed sort fields.
- Apply per-user/workspace and per-IP rate limits where appropriate.
- Use CSRF protection if cookie-based auth is used.
- Restrict CORS to configured origins.
- Enforce object-level authorization on every endpoint, including nested resource IDs and bulk operations.
- Apply request-size and upload-size limits.
- Require idempotency on upload finalization, indexing/extraction jobs, answer generation/regeneration and export creation.
- Use optimistic concurrency for transcription revisions, answer edits and approval pointers.
- Do not accept arbitrary storage keys, model names, provider URLs, SQL fragments or callback URLs from untrusted clients.
- Workers must re-check workspace ownership, source version and tombstone state immediately before committing output.
