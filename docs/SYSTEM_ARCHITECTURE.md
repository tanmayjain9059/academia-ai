# System architecture

Status: proposed logical architecture. Vendor and instance sizing remain subject to benchmarks and cost approval.

## 1. Architecture principles

- Cloud-first, multi-tenant, modular monolith first.
- Separate asynchronous workers from latency-sensitive API requests.
- Persist all durable state in managed services; compute instances are disposable.
- Keep inference providers replaceable behind a small adapter contract.
- Use one authoritative relational database for ownership, workflow state and metadata.
- Use object storage for original files and large derived artifacts.
- Add distributed components only where the workload requires them.

## 2. Logical components

1. Web client: React + TypeScript; authenticated workspace, uploads, extraction review, answer editor and export UI.
2. API: Python + FastAPI; authentication integration, authorization, validation, domain operations, presigned upload/download orchestration.
3. PostgreSQL: tenants/users, subjects, documents, extraction records, questions, answer versions, source references, jobs, usage and audit metadata.
4. Object storage: originals, rendered page previews, extracted media, generated PDFs and temporary processing artifacts.
5. Durable queue: job dispatch for ingestion, OCR, embedding, answer generation and export; requires at-least-once-safe processing, visibility/lease extension, bounded retries and dead-letter handling.
6. CPU workers: parsing, conversion, page rendering, OCR pre/post-processing, chunking and PDF generation.
7. GPU/model workers or external inference endpoint: OCR/VLM, embeddings/reranker and answer generation according to benchmarked demand.
8. Vector retrieval: initially PostgreSQL with pgvector plus PostgreSQL full-text search; evaluate a separate vector database only when measured scale or query behavior justifies it.
9. Observability: structured logs, metrics, traces, error reporting and cost/usage events with sensitive content redacted.

## 3. Request and ingestion flow

1. Client requests an upload intent for a subject and file category.
2. API authenticates the user, checks tenant quota and creates a pending document record.
3. API returns a short-lived, object-specific signed upload URL.
4. Client uploads directly to private object storage (for R2, evaluate signed PUT; presigned HTML form POST is not supported).
5. API finalizes the upload; a worker verifies object existence, size, checksum and actual file signature.
6. An ingestion job is created transactionally with an outbox event or equivalent reliable dispatch.
7. Worker routes the file by detected format and extracts native structure first; Docling is the leading normalization candidate, with specialist parser fallbacks for fidelity gaps.
8. Scanned/image pages are sent to OCR; complex layout pages can be escalated selectively.
9. Normalized blocks retain page/slide/sheet coordinates, extraction method, confidence and original asset reference.
10. Question detection creates candidate question records; reference ingestion creates structured chunks.
11. UI shows job status and source preview for user verification.
12. Verified source content is indexed for retrieval.

## 4. Answer generation flow

1. API verifies ownership of question, subject and selected sources.
2. Retrieval applies tenant, subject, source-type and optional unit filters before similarity ranking.
3. Hybrid retrieval combines lexical search and vector similarity; optional reranking follows benchmark evidence.
4. Prompt construction includes question text, answer mode, marks, syllabus profile and selected source passages with provenance.
5. Inference adapter calls the chosen model with bounded context, timeout and token budget.
6. Answer validator checks schema, citation identifiers, source coverage and unsupported claims where feasible.
7. Persist answer version, model/provider/version, prompt template version, source references, token usage and status.
8. UI displays answer and citations; user can edit, approve or regenerate without changing the question.

## 5. Reliability and concurrency

- Jobs have explicit states: queued, running, succeeded, failed, cancelled.
- Every job has tenant_id, idempotency key, attempt count, timestamps and safe error category.
- Retries are bounded and limited to transient failures. Invalid files and policy failures are terminal.
- Worker operations are idempotent; duplicate queue delivery must not duplicate user-visible records.
- Use database transactions for state transitions and related writes.
- Long jobs heartbeat and extend queue visibility/leases; a worker crash must eventually make the job eligible for retry.
- Use a transactional outbox or equivalent to avoid committing a job record while losing its queue dispatch.
- Rate-limit expensive work per user and globally.
- GPU workers use bounded concurrency and backpressure; queue rather than exhaust memory.
- Use per-provider circuit breakers/timeouts and a controlled fallback policy.
- Do not expose raw stack traces, prompts, secrets or other tenants' content in errors/logs.

## 6. Initial deployment shape

Start with:
- One web frontend deployment.
- One API deployment.
- One PostgreSQL database.
- One private object-storage bucket.
- One durable queue and a small CPU worker deployment.
- A separately deployable inference endpoint/provider.

Keep API and workers in a modular codebase initially. Avoid Kubernetes and microservices until metrics show a clear need. A single shared GPU endpoint may serve the initial beta, but enforce queueing, request limits and cost budgets.

## 7. Scaling path

- Scale API horizontally; keep sessions and job state out of process memory.
- Scale CPU workers by queue depth and job duration.
- Scale inference independently by GPU utilization, queue wait, token throughput and cost.
- Partition or archive large event/usage tables only after observing growth.
- Add dedicated vector infrastructure only after query benchmarks justify it.
- Add tenant-level quotas, plan entitlements and fair scheduling before opening broad public access.

## 8. Key architecture risks

- OCR/model quality on mixed scans, handwriting, math and tables.
- GPU cost and cold-start latency.
- Document parser vulnerabilities and malicious embedded content.
- Tenant leakage through retrieval filters, cache keys, object URLs or asynchronous jobs.
- Large uploads and archive bombs.
- Citation drift when source documents are replaced or reprocessed.
- PDF layout fidelity for long answers, equations and diagrams.
- Provider-specific rate limits and model availability.
