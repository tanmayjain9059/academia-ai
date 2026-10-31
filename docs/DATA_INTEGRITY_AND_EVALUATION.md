# Data integrity, tenant isolation and evaluation specification

Research checked: 2026-10-31

Status: architecture-level requirements. This document is a design and test contract, not implementation or a claim that controls are already deployed.

## 1. Canonical ownership boundary

Use `workspace_id` as the single canonical tenant key. A workspace is the tenant boundary, including when the first release creates one personal workspace per student. Avoid storing both `tenant_id` and `workspace_id` as independently mutable ownership columns.

Every workspace-owned relational row carries `workspace_id`, including documents, processing runs, pages, blocks, questions, transcription versions, chunks, embeddings, answers, answer versions, answer sources, jobs, exports, usage and audit events where applicable.

Enforce ownership in four places:
1. API/domain layer: derive the actor from a verified identity token; resolve workspace membership server-side; never trust a client-supplied workspace or owner field.
2. Database: tenant-scoped query predicates, composite foreign keys and RLS as defense in depth.
3. Object storage: private bucket, opaque non-user-controlled keys, short-lived object-specific signed URLs issued only after resource authorization.
4. Async processing: queue messages contain opaque IDs; workers reload the job and resource from the database, verify matching workspace ownership, and use that workspace context for every query and storage operation.

A workspace ID in a request, job message or object key is not proof of authorization.

## 2. PostgreSQL RLS design constraints

- Enable RLS on all workspace-owned tables exposed to application roles.
- Create explicit policies for SELECT, INSERT, UPDATE and DELETE. Use `USING` to constrain existing/visible rows and `WITH CHECK` to constrain inserted or resulting rows.
- Default to deny when no applicable policy exists.
- Use a dedicated runtime role that does not own protected tables and does not have `BYPASSRLS`. Keep schema migration ownership separate from application runtime credentials.
- Consider `FORCE ROW LEVEL SECURITY` on protected tables, while remembering superusers and `BYPASSRLS` roles still bypass it.
- Set tenant context transaction-locally on every database transaction before tenant-scoped statements. With transaction pooling, do not rely on session-level state. The implementation can use a parameterized `set_config('app.workspace_id', workspace_id, true)` call inside the transaction.
- Missing tenant context must produce no visible tenant rows, not a global query.
- Never reuse a connection with tenant context left behind. Test pooled connection reuse by alternating workspace A, workspace B and no-context transactions.
- Keep administrative maintenance in a separate, narrowly scoped path with audited access; do not use a privileged service credential for ordinary user requests.
- RLS is not a replacement for object-level authorization, composite relational constraints or application tests.

PostgreSQL documents that table owners normally bypass RLS unless FORCE is used, and roles with BYPASSRLS/superuser privileges always bypass it. PostgreSQL policies distinguish USING and WITH CHECK. SET LOCAL applies only for the current transaction; this is the safe direction for transaction-pooled connections.

## 3. Relational integrity for tenant boundaries

Use composite unique keys and composite foreign keys to prevent accidental cross-workspace relationships even when IDs are globally unique.

Illustrative pattern (not a complete migration):

```sql
CREATE TABLE subjects (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  UNIQUE (workspace_id, id)
);

CREATE TABLE documents (
  id uuid PRIMARY KEY,
  workspace_id uuid NOT NULL,
  subject_id uuid NOT NULL,
  UNIQUE (workspace_id, id),
  FOREIGN KEY (workspace_id, subject_id)
    REFERENCES subjects (workspace_id, id)
);
```

Apply the same ownership-preserving pattern to document runs/pages/blocks, questions/transcriptions, reference chunks/embeddings, answers/versions/sources, jobs and exports. Add indexes on referencing columns where query and delete paths need them. Use NOT NULL for mandatory ownership and parent references. Do not rely on CHECK constraints that query other rows; PostgreSQL does not guarantee cross-row invariants through such CHECK expressions.

## 4. Immutable source and version lifecycle

Keep five artifact classes distinct:

1. **Original upload** — exact uploaded bytes, SHA-256, byte length, detected format, upload metadata and immutable object key. Never transform or overwrite this object. A corrected upload creates a new document record or explicit new source revision; it does not replace bytes behind an existing identity.
2. **Processing run** — append-only run metadata: parser/OCR/model versions, configuration, start/end time, status and safe failure category. Reprocessing creates a new run.
3. **Raw extraction** — parser output and normalized blocks tied to one processing run. Never overwrite raw text or coordinates. New parser versions create new run-scoped output.
4. **Verified transcription** — append-only revisions with editor, timestamp, parent revision, exact text and verification status. UI clearly distinguishes extracted text from verified text. A user edit must not mutate original bytes or raw extraction.
5. **Answer/export snapshot** — every answer version records its exact question-transcription version, answer mode, subject profile/prompt version, model/provider version and source block/chunk IDs. Exports record exact question and answer version IDs plus template/version metadata.

Question identity and source location are independent of question text. A correction changes a transcription revision, not the question's identity. An answer regeneration creates a new answer version and never mutates the question or its prior answers.

### Deletion and retention

Deletion is a lifecycle, not a single SQL DELETE:
- Mark a document as deletion-requested/tombstoned so it disappears from normal UI and cannot be used for new retrieval or generation.
- Cancel or invalidate queued work for the document; workers re-check tombstone and source revision before writing output.
- Purge original and derived objects, OCR previews, page renders, extracted media, chunks, embeddings, caches and dependent exports according to the approved retention policy.
- Preserve only the minimum tombstone/audit evidence required to prevent accidental resurrection and prove deletion processing; do not retain source text in audit logs.
- Handle shared/deduplicated blobs with reference counting and race-safe transactional deletion.
- Define backup expiry and restore-time deletion replay before public launch.
- Existing answer citations should resolve to an explicit “source deleted/unavailable” state; do not silently retarget them to a different document version.

Retention duration, backup purge windows and legal hold behavior remain owner decisions.

## 5. Upload and parser trust boundary

Use defense in depth, consistent with OWASP upload guidance:
- Allowlist only required extensions; normalize and constrain filenames, but never use user filenames as storage paths.
- Treat client MIME as untrusted. Check extension, MIME and file signature together; no single check is sufficient.
- Enforce compressed and decompressed byte limits, archive entry count/depth, page count, dimensions, processing time and memory budgets.
- Reject or explicitly quarantine encrypted, malformed, unsupported and suspicious files.
- Store originals privately outside the application web-serving path.
- Run parsers, converters, OCR and archive inspection in isolated unprivileged workers with restricted filesystem and network access.
- Never execute macros, embedded scripts or arbitrary archive contents.
- Keep antivirus/CDR as optional additional signals, not substitutes for sandboxing and parser patching.
- Do not submit private academic documents to public scanning services without an explicit privacy review and user consent.

## 6. Benchmark corpus design

Create a private, versioned benchmark with document-level separation between development, tuning and held-out evaluation sets. Do not split pages from the same original document across train/dev/test because near-duplicate layouts can inflate results.

Stratify the corpus by:
- Born-digital vs scanned PDF.
- Image format and camera-photo quality (rotation, skew, blur, shadows, perspective).
- English and selected Indian scripts/languages; report each separately.
- Single/multi-column layout, tables, diagrams, equations and mixed content.
- DOCX/PPTX/XLSX features: tables, images, slide notes, merged cells, formulas, hidden rows/columns and reading order.
- Question structures: marks, nested numbering, OR choices, subparts, headers/footers and continuation across pages.
- Handwriting as a separate exploratory category; do not blend it into printed-text averages.
- Difficult/negative inputs: corrupted, encrypted, MIME-mismatched, malformed and oversized files.

Use non-sensitive documents with permission for evaluation. Keep copyrighted or personal source documents outside public Git and restrict access to the benchmark corpus. Maintain human-verified ground truth, annotation instructions, corpus version and reviewer disagreements.

## 7. Metrics and release gates

Report metrics by format, language/script and difficulty stratum, not only one pooled average.

| Stage | Measures |
|---|---|
| OCR | Character error rate (CER), word error rate (WER), exact text match for question regions |
| Layout | Reading-order accuracy, block/region detection, table cell structure and coordinate alignment |
| Question extraction | Question boundary precision/recall/F1, hierarchy accuracy, numbering accuracy, marks exact-match rate |
| Retrieval | Recall@k, MRR or nDCG, correct-source retrieval rate, tenant-filter leakage tests |
| Answering | Claim-level source support, citation precision/recall, factual error rate, syllabus-scope compliance, answer completeness by marks rubric |
| Export | Text extraction regression, page overflow/clipping, numbering continuity, image/crop fidelity and print review |
| Operations | p50/p95 latency, throughput, retry/failure rate, memory/CPU/GPU peak, cost per page and cost per accepted answer |

CER/WER alone are insufficient: a transcription can score well while losing reading order, table structure or question hierarchy. Review structural fidelity and task-level correctness separately.

Before selecting a parser/model or releasing a pipeline stage:
1. Freeze the held-out corpus version and evaluation script.
2. Establish a baseline with a simple/native parser and human-reviewed samples.
3. Define numeric pass thresholds from product risk and observed baseline; do not invent thresholds before the first measurement.
4. Compare candidates on the same hardware, corpus and preprocessing.
5. Record model/parser version, configuration, license, latency, resource usage and cost.
6. Inspect worst-case failures manually, especially lost negations, numbers, units, marks, formula symbols and sub-question boundaries.
7. Require regression tests for each accepted failure fix.

## 8. Tenant-isolation test matrix

Automated integration tests must prove deny behavior, not just successful owner access.

| Surface | Required negative test |
|---|---|
| REST/API | Workspace B cannot list, read, mutate or delete Workspace A resources by guessing IDs |
| SQL/RLS | No context sees no tenant rows; A context cannot read/write B rows; update cannot move a row to B |
| Pooling | Reuse the same pooled connection across A, B and no-context transactions without context leakage |
| Relational constraints | Insert a child in workspace A referencing a parent in workspace B is rejected |
| Object storage | A cannot obtain a signed URL or fetch an object belonging to B |
| Queue | Forged or stale job payload cannot cause a worker to process a different workspace's resource |
| Retrieval | Vector and lexical search results never include another workspace's chunks |
| Cache | Cache keys include workspace and relevant source/version scope; cross-workspace collisions are tested |
| Export | A cannot inspect, cancel or download B's export, including after URL expiry |
| Deletion | Tombstoned/deleted sources cannot be re-indexed by stale workers or reappear after retry |

Run tests using the actual least-privilege runtime database role, not only the migration/table-owner role.

## 9. Research references checked 2026-10-31

- PostgreSQL row security: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- PostgreSQL CREATE POLICY (USING/WITH CHECK): https://www.postgresql.org/docs/current/sql-createpolicy.html
- PostgreSQL SET LOCAL: https://www.postgresql.org/docs/current/sql-set.html
- PostgreSQL constraints and composite keys: https://www.postgresql.org/docs/current/ddl-constraints.html
- Supabase RLS and policy testing guidance: https://supabase.com/docs/guides/database/postgres/row-level-security
- PgBouncer transaction pooling: https://www.pgbouncer.org/features.html
- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- OCR evaluation survey (HIP 2021): https://doi.org/10.1145/3476887.3476888
- OCR benchmark comparison (IEEE INMIC 2024): https://doi.org/10.1109/INMIC64792.2024.11004392
