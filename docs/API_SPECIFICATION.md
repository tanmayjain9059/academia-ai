# API specification

Status: proposed resource contract. Exact payload schemas and OpenAPI definitions will be authored alongside implementation.

## 1. Conventions

- Prefix: /api/v1.
- JSON request/response except binary upload/download handled through object storage.
- Authentication required except health and public metadata routes.
- Tenant/workspace scope is derived from authenticated membership, never trusted from an arbitrary client-supplied tenant ID.
- Use opaque IDs, pagination, request IDs and idempotency keys for mutation endpoints.
- Return stable error codes; never leak stack traces or private resource existence across tenants.

## 2. Planned resources

### Workspaces and academics
- POST /workspaces
- GET /workspaces
- GET /workspaces/{workspace_id}
- POST /workspaces/{workspace_id}/semesters
- POST /subjects
- GET /subjects
- GET /subjects/{subject_id}

### Documents
- POST /documents/upload-intents
- POST /documents/{document_id}/finalize
- GET /documents
- GET /documents/{document_id}
- DELETE /documents/{document_id}
- POST /documents/{document_id}/process
- GET /documents/{document_id}/processing-runs
- GET /documents/{document_id}/pages
- GET /documents/{document_id}/blocks

Upload flow: API authorizes and creates a pending record, returns a short-lived signed upload URL, client uploads directly to private object storage, then finalization verifies object metadata and schedules processing. The server must independently verify file signature and actual size.

### Questions and transcription review
- POST /documents/{document_id}/question-extractions
- GET /questions
- GET /questions/{question_id}
- POST /questions/{question_id}/transcriptions
- GET /questions/{question_id}/transcriptions
- POST /questions/{question_id}/verify

Verification creates a new transcription version. It does not edit the source file or raw extraction.

### Reference retrieval and answers
- POST /answers/generate
- GET /questions/{question_id}/answers
- GET /answers/{answer_id}
- GET /answers/{answer_id}/versions
- PATCH /answers/{answer_id}/draft
- POST /answers/{answer_id}/regenerate
- POST /answer-versions/{version_id}/approve
- GET /answer-versions/{version_id}/sources

Generation requests include question ID, selected source IDs, answer mode, marks/length target and optional subject-profile settings. API validates that every resource belongs to the caller's workspace.

### Jobs and exports
- GET /jobs/{job_id}
- POST /jobs/{job_id}/cancel
- POST /exports
- GET /exports/{export_id}
- GET /exports/{export_id}/download

Download URLs are short-lived and object-specific. Exports are private and workspace-scoped.

## 3. Job contract

Expose status, progress stage, safe user-facing message, created/updated timestamps and retryability. Do not expose raw worker tracebacks, prompts or internal provider credentials.

## 4. Error contract

Use a consistent envelope with code, message, request_id and optional field_errors. Expected codes include:
- AUTH_REQUIRED
- FORBIDDEN
- NOT_FOUND
- VALIDATION_ERROR
- UNSUPPORTED_FILE
- FILE_TOO_LARGE
- FILE_SIGNATURE_MISMATCH
- QUOTA_EXCEEDED
- JOB_CONFLICT
- RATE_LIMITED
- MODEL_UNAVAILABLE
- PROCESSING_FAILED
- EXPORT_FAILED

Do not reveal whether a private resource exists when the caller is not authorized to access it.

## 5. API security requirements

- Validate request bodies and query parameters.
- Apply per-user and per-IP rate limits where appropriate.
- Use CSRF protection if cookie-based auth is used.
- Restrict CORS to configured origins.
- Enforce object-level authorization on every endpoint.
- Apply request-size and upload-size limits.
- Require idempotency on upload finalization and expensive job creation.
- Do not accept arbitrary storage keys, model names, provider URLs or callback URLs from untrusted clients.
