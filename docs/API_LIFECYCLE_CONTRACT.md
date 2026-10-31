# API lifecycle and mutation contracts

Status: architecture contract for API implementation. Exact JSON/OpenAPI schemas will be generated during implementation; this document defines behavior that must not vary by endpoint.

## 1. Resource and authorization model

- API prefix: `/api/v1`.
- `workspace_id` is the canonical ownership boundary.
- The authenticated principal is derived from a verified access token. Workspace membership and role are loaded server-side for each request or from a safely invalidated authorization cache.
- A workspace ID in a path is a requested scope, not proof of membership.
- Every resource lookup is constrained by workspace membership and resource ownership. Return a consistent NOT_FOUND response for inaccessible private resources to avoid resource enumeration.
- Child resources inherit workspace ownership. The database enforces matching workspace ownership through composite foreign keys; the API verifies the full parent-child relationship.
- Workers do not trust workspace IDs in queue payloads. They load the persisted job and resource, then verify ownership and state.

## 2. Common request/response conventions

- JSON for control-plane APIs; direct binary upload/download uses private object storage.
- Use opaque UUIDs or equivalent non-sequential identifiers. Never expose object-store keys as public resource identifiers.
- Paginate collections with a stable cursor. Define deterministic ordering (created_at plus id) and avoid offset pagination for rapidly changing job/document collections.
- Return a request ID on every response. Error envelope: `code`, safe `message`, `request_id`, optional `field_errors`.
- Use UTC timestamps in ISO 8601.
- Use ETags or explicit version numbers for mutable resource heads. Mutations must include `If-Match` or an expected version when concurrent edits could overwrite each other; respond with a conflict on stale versions.
- Require an `Idempotency-Key` for upload finalization, extraction/indexing jobs, answer generation/regeneration and export creation. Scope keys by workspace, actor and operation. Store request hash and final response/resource mapping; same key with a different request body is a conflict.
- Do not allow clients to choose arbitrary model IDs, provider endpoints, object keys, parser commands or callback URLs. Client may select an exposed answer mode and bounded product options only.

## 3. Resource lifecycle contracts

### Workspace, semester and subject

- Workspace creation creates the initial owner membership atomically.
- Workspace membership and role checks are server-side. MVP may expose only owner role and one-person workspaces; do not build collaboration UI prematurely.
- Semester and subject creation/update/deletion are scoped to workspace. Deleting a semester with subjects must require explicit cascade/reassignment behavior; never silently cascade academic content.
- Subject profiles are versioned or snapshot into each answer generation so later profile edits do not rewrite prior answer context.

### Document upload and processing

1. `POST /documents/upload-intents` accepts subject, category, filename, declared content type and expected size. It validates workspace/subject ownership and quota.
2. API creates a pending document/upload record with an opaque object key and expected constraints. It returns a short-lived signed PUT URL and required headers.
3. Client uploads directly to a private bucket. The API never accepts an arbitrary object key from the browser.
4. `POST /documents/{id}/finalize` requires an idempotency key. Server verifies object existence, actual size, checksum if supplied, file signature and detected type. It does not trust client MIME or checksum alone.
5. On successful validation, finalize atomically records immutable original metadata and creates a processing job/outbox event. Duplicate finalize requests return the original result; a different payload with the same key conflicts.
6. Processing creates a new immutable processing run. Reprocessing never overwrites prior extraction; it creates a new run and a new set of blocks.
7. Pages/blocks are addressable only within their document and processing run. Clients cannot supply block IDs to another document.
8. Processing failures expose safe categories and remediation guidance, not tracebacks or raw parser output.
9. A document in deletion-requested/deleted state cannot be finalized, processed, indexed or selected for new answer generation.

### Question extraction and transcription

- Question extraction is a job scoped to a specific document processing run and extraction configuration.
- Candidate questions reference exact source block IDs and source locations.
- Question identity is stable across transcription revisions and reprocessing where a reliable matching decision exists. If matching is ambiguous, surface a new candidate for user review instead of silently merging identities.
- Raw extraction text is immutable. User correction creates an append-only transcription revision with editor, timestamp, parent revision and verification state.
- Verification is an explicit state transition on a specific revision; it must not overwrite source blocks or raw extraction.
- Concurrent edits use expected revision IDs; stale edits return conflict and do not discard either version.
- A question's current verified transcription is a pointer to a revision, not mutable text stored on the question row.

### Reference sources and indexing

- Question-source documents and reference-source documents are separate categories in the product workflow, even though they share secure upload and document processing infrastructure.
- Reference indexing is a separate idempotent job bound to a specific processing run and chunking/embedding configuration.
- Each chunk and embedding stores workspace, subject, source document, processing run, source block range and model/version metadata.
- Re-indexing creates a new index version. Old answers retain their exact source IDs and provenance.
- Retrieval must filter workspace and subject before ranking. User-selected source IDs are re-authorized and checked against the question's workspace/subject before generation.

### Answer generation, editing and approval

- Generation input pins the question transcription revision, answer mode, subject-profile version, selected reference/index versions and requested output constraints.
- API rejects a question and source set that do not belong to the same authorized workspace and permitted subject scope.
- Every generation creates a new immutable answer version with model/provider/version, prompt-template version, source IDs, token/cost/latency metadata and validation result.
- An answer is a stable logical resource whose current draft/approved pointers reference immutable answer versions.
- Editing creates a new user-authored answer version with a parent version and editor metadata. Do not PATCH the content of an already persisted version.
- Approval is a pointer/state transition to a specific immutable version and uses optimistic concurrency.
- Regeneration creates a new version; it never replaces the previous answer or silently changes the question transcription.
- Citation links must resolve to the exact source processing run and block/chunk. If a source is deleted, the citation becomes explicitly unavailable; it must not retarget to a newer processing run.

### Jobs

- A job is a persisted workspace-owned resource. Queue messages contain only job ID and minimal routing metadata.
- State machine: `queued -> running -> succeeded | failed | cancelled`. Retryable failure may transition back to queued through a recorded retry attempt. Terminal states do not silently reopen; explicit retry creates a new attempt or job linked to the original.
- Every state transition is conditional on expected state/version and worker lease token. A worker whose lease expired cannot commit results after another worker has reclaimed the job.
- Cancellation is cooperative. A cancel request sets a cancellation flag; workers check it at safe boundaries and before committing outputs. Cancellation does not imply already committed immutable artifacts are erased.
- Progress is best-effort and monotonic within a stage; clients must not interpret percentages as guaranteed completion time.
- Queue delivery is at-least-once. Workers must be idempotent, and database uniqueness constraints prevent duplicate user-visible outputs.
- Retry only transient errors with bounded exponential backoff and jitter. Invalid files, authorization failures and policy violations are terminal.
- Expose safe status, stage, attempt count, timestamps and user-facing error category. Never expose stack traces, prompts, private content or credentials.

### Exports and downloads

- Export creation pins an explicit ordered list of question IDs, transcription revision IDs, answer version IDs and template version.
- The export job validates all referenced records belong to the same workspace and remain exportable.
- An export is an immutable snapshot. Subsequent answer edits do not mutate an existing PDF.
- Download endpoint re-authorizes the caller and issues a short-lived object-specific signed URL. Do not persist or log the signed URL as a durable resource field.
- Expired exports may be regenerated as a new export resource. Existing export identity and contents remain stable.
- Deletion and retention policies must specify whether exports are purged when any included source or answer is deleted.

## 4. Deletion and stale worker protection

- Document deletion first sets a tombstone/deletion-requested state and removes the document from normal listing and new retrieval/generation.
- All workers must re-check source state and source processing-run identity immediately before committing output.
- A stale worker may not recreate blocks, chunks, embeddings or exports after deletion has begun.
- Purge is asynchronous and idempotent. Track purge status for original objects, derived assets, chunks/vectors, caches and dependent exports.
- Answer history retains a source-deleted marker where required for integrity; no source text should be copied into audit logs.
- Account deletion is a higher-level workflow covering workspace ownership transfer or workspace deletion, provider identity deletion, object purge, database records, caches, exports, backups and legal retention exceptions. Exact timelines require owner/legal approval.

## 5. Endpoint inventory and gaps to resolve in OpenAPI

The resource inventory should include these route families (final REST shapes may change during implementation):

| Resource | Required operations |
|---|---|
| Workspaces | create, list, read, update settings, request deletion |
| Semesters | create, list, read, update, delete with explicit dependency behavior |
| Subjects | create, list, read, update, delete/archive |
| Documents | upload intent, finalize, list, read, request deletion, reprocess |
| Processing runs | list, read, inspect safe diagnostics |
| Pages/blocks | paginated read scoped to document + run |
| Question extraction | create job, list candidates, read candidates |
| Questions | list/filter, read, update ordering/group metadata where allowed |
| Transcriptions | list revisions, create revision, verify revision, select current revision |
| Reference indexing | create/rebuild index job, list index versions, inspect chunks |
| Answers | generate, list by question, read logical answer |
| Answer versions | list, read, create edited version, approve/select current |
| Jobs | read, cancel, retry where policy permits |
| Exports | create, read, list, download, expire/delete |
| Account/privacy | export personal data, request account/workspace deletion, inspect deletion status |

Do not expose a general endpoint that accepts arbitrary SQL filters, storage keys or model/provider names.

## 6. Acceptance tests before API implementation is considered complete

- Every endpoint rejects a caller who is not a workspace member, including nested resource IDs.
- Cross-workspace parent-child IDs are rejected even when both IDs are syntactically valid.
- Repeated idempotency key with identical body returns the same logical result; different body returns conflict.
- Concurrent transcription edits do not silently overwrite each other.
- Concurrent answer approvals use optimistic concurrency and produce one unambiguous current pointer.
- Duplicate queue delivery creates no duplicate output.
- Expired worker lease cannot commit.
- Cancelled/deleted source is not resurrected by a stale worker.
- Signed URLs are short-lived, object-specific and never exposed to another workspace.
- Export snapshot remains unchanged after later answer edits.
- Deleted sources resolve to an explicit unavailable citation state.
- Error responses do not disclose existence or contents of unauthorized resources.
