# Technical research and provisional decisions

Research checked: 2026-10-31

This document records evidence-backed candidates, not final procurement approvals.

## Document ingestion

Docling is the leading candidate for a common normalization layer. Its supported formats include PDF, modern and legacy Office (legacy formats require LibreOffice), OpenDocument, EPUB, markup/text, raster images, and optional audio/video paths. Feature-level fidelity must be tested; format support does not guarantee preservation of notes, formulas, charts, merged cells or coordinates. Retain specialist parsers where they preserve more faithful structure.

Docling code is MIT-licensed. PaddleOCR is Apache-2.0 at repository level; individual model weights and dependencies still require review. PaddleOCR v3.7.0 announced PP-OCRv6. Treat PP-OCRv6 as a benchmark candidate, with PP-StructureV3/PaddleOCR-VL for selective escalation.

## Database

Keep PostgreSQL + pgvector + full-text search as the initial retrieval architecture. Supabase documents pgvector support and lists Mumbai (AWS ap-south-1) as an available specific region. This makes it a concrete India-first candidate, not an automatic selection. Verify exact-plan extension/index support, pooling, backups, RLS, region, subprocessors and total cost.

## Object storage

Keep private S3-compatible storage behind an adapter; R2 is a candidate. R2 documents S3 compatibility and presigned GET, HEAD, PUT and DELETE. Presigned HTML form POST is not supported. Use short-lived object-specific PUT URLs, narrow CORS, and independent server-side size/signature/checksum validation after upload. Signed URLs are bearer credentials and must not be logged.

R2 pricing documentation currently lists standard storage at USD 0.015/GB-month and no internet egress charge, with separate operation pricing. This is a dated snapshot, not a forecast.

## Queue

Require durable dispatch, at-least-once-safe idempotent workers, visibility/lease extension, bounded retries and a DLQ. SQS documents at-least-once delivery for standard queues; messages can be redelivered if processing is not acknowledged before visibility expires. QStash supports retries and FIFO queues, but its queue behavior serializes messages within a queue while a message is retried. Compare SQS, managed Redis worker queues and QStash against parallelism, region, costs and operations before selecting.

Use a transactional outbox or equivalent reliable database-to-queue dispatch. Queue payloads should contain opaque resource/version IDs, not document text.

## Authentication

Keep identity replaceable. FastAPI should validate signed access tokens and derive workspace authorization from server-side membership records. Better Auth supports organizations and roles but is a TypeScript framework; adding it would require an explicit decision about an additional auth runtime. Clerk documents Python backend token verification; Supabase Auth is another candidate if Supabase Postgres is selected. Compare MFA, recovery, key rotation, revocation, export/deletion, India data handling, subprocessors and pricing.

## Candidate matrix

| Layer | Candidate | Status |
|---|---|---|
| Web | React + TypeScript | Proposed |
| API | Python + FastAPI | Proposed |
| Normalization | Docling | Benchmark candidate |
| OCR | PaddleOCR PP-OCRv6 | Benchmark candidate |
| Layout escalation | PP-StructureV3 / PaddleOCR-VL | Benchmark candidate |
| Database | Managed PostgreSQL + pgvector + FTS | Provider open; Supabase Mumbai candidate |
| Object storage | Private S3-compatible; R2 candidate | Provider open |
| Queue | Durable queue with leases + DLQ | Vendor open |
| Auth | Managed OIDC/JWT provider | Provider open |
| Inference | Replaceable hosted/self-hosted endpoint | Provider/model open |

## Official references checked 2026-10-31

- Docling formats: https://docling-project.github.io/docling/usage/supported_formats/
- Docling license: https://github.com/docling-project/docling
- Docling OCR support: https://docling-project.github.io/docling/concepts/OCR_native/
- PaddleOCR license and releases: https://github.com/PaddlePaddle/PaddleOCR and https://github.com/PaddlePaddle/PaddleOCR/releases
- Supabase pgvector: https://supabase.com/docs/guides/ai/vector-columns
- Supabase regions: https://supabase.com/docs/guides/platform/regions
- R2 S3 compatibility: https://developers.cloudflare.com/r2/get-started/s3/
- R2 presigned URLs: https://developers.cloudflare.com/r2/api/s3/presigned-urls/
- R2 pricing: https://developers.cloudflare.com/r2/pricing/
- SQS visibility timeout: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- QStash queues/retries: https://upstash.com/docs/qstash/features/queues and https://upstash.com/docs/qstash/features/retry
- Better Auth organizations: https://better-auth.com/docs/plugins/organization
- Clerk Python backend: https://clerk.com/articles/how-to-add-authentication-to-a-python-backend
- Clerk JWT verification: https://clerk.com/docs/guides/sessions/manual-jwt-verification
