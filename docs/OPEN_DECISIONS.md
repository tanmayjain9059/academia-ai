# Open decisions and architecture gates

Status: active decision register. No item below should be silently treated as approved.

| ID | Decision | Current recommendation | Evidence/owner action needed |
|---|---|---|---|
| D-01 | Initial launch geography | India-first assumption for evaluation only | Confirm legal/data residency and target users |
| D-02 | Frontend hosting | Static frontend on a CDN | Compare build/deploy workflow and free-tier limits |
| D-03 | API/worker hosting | Managed container platform | Benchmark region, persistent networking, queue and costs |
| D-04 | PostgreSQL provider | Managed Postgres; pgvector required; Supabase Mumbai is a concrete candidate | Verify exact-plan extension/index support, pooling, backups, RLS, region and total cost |
| D-05 | Object storage | Private S3-compatible storage; R2 candidate | Test signed PUT/multipart, CORS, lifecycle, bucket location and data processing terms |
| D-06 | Queue | Durable managed queue with visibility leases and DLQ; SQS is a reference implementation | Compare SQS, managed Redis worker queue and QStash for parallelism, retries, region, cost and ops |
| D-07 | Inference hosting | Hugging Face for experimentation; production provider undecided | Benchmark model quality, concurrency, privacy, cold start and cost |
| D-08 | OCR model | PaddleOCR PP-OCRv6 baseline candidate; PP-Structure/VL for escalation | Benchmark scans, tables, equations, mixed language and handwriting separately; review model licenses |
| D-09 | Answer LLM | Qwen and other open-weight models as candidates | Build human-reviewed academic benchmark; verify license and serving |
| D-10 | Embedding model | BGE-M3 and multilingual-e5 candidates | Compare retrieval recall, language support, dimensions, cost and latency |
| D-11 | Authentication | Managed OIDC/JWT provider; Clerk and Supabase Auth candidates | Compare FastAPI integration, MFA, user export/deletion, India data handling, pricing and lock-in |
| D-12 | Pricing model | Free/limited beta first; usage-aware paid tier may follow | Agree budget ceiling and expected free allowance |
| D-13 | Data retention | User-controlled deletion; retention period undecided | Decide backups, deleted-object purge window and legal requirements |
| D-14 | Collaboration | Excluded from MVP | Revisit only after single-user workflow is proven |
| D-15 | Audio/video | Later phase | Revisit after document pipeline quality is stable |

## Required benchmark corpus

Owner should provide or approve a representative set of non-sensitive documents:
- Digital and scanned PDFs.
- Photos of question papers.
- DOCX/PPTX/XLSX examples.
- Tables, equations, diagrams and nested question numbering.
- English and relevant Indian-language examples.
- Reference material and expected answer examples.

Do not upload copyrighted or personal academic files into public benchmark repositories. Store private evaluation assets outside Git and restrict access.

## Required owner decisions before implementation

1. Confirm target audience and launch geography.
2. Agree initial hosting/inference budget ceiling.
3. Confirm whether the first beta is invite-only.
4. Approve the supported MVP file formats.
5. Approve source-only vs combined-knowledge answer behavior.
6. Approve data retention and deletion expectations.
7. Approve final model/provider selection after benchmark results.

See [Technical research and provisional decisions](TECHNICAL_DECISIONS.md) for evidence checked on 2026-10-31. That document is research, not a provider procurement approval.
