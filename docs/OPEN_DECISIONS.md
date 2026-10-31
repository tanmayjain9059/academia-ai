# Open decisions and architecture gates

Status: active decision register. No item below should be silently treated as approved.

| ID | Decision | Current recommendation | Evidence/owner action needed |
|---|---|---|---|
| D-01 | Initial launch geography | India-first assumption for evaluation only | Confirm legal/data residency and target users |
| D-02 | Frontend hosting | Static frontend on a CDN | Compare build/deploy workflow and free-tier limits |
| D-03 | API/worker hosting | Managed container platform | Benchmark region, persistent networking, queue and costs |
| D-04 | PostgreSQL provider | Managed Postgres; pgvector required | Verify extension/version, pooling, backups, RLS support and limits |
| D-05 | Object storage | S3-compatible private storage; R2 candidate | Test signed URLs, CORS, lifecycle, region and data processing terms |
| D-06 | Queue | Managed Redis or managed durable queue | Compare delivery semantics, persistence, retries and operational burden |
| D-07 | Inference hosting | Hugging Face for experimentation; production provider undecided | Benchmark model quality, concurrency, privacy, cold start and cost |
| D-08 | OCR model | PaddleOCR family as baseline candidate | Benchmark scans, tables, equations, mixed language and handwriting separately |
| D-09 | Answer LLM | Qwen and other open-weight models as candidates | Build human-reviewed academic benchmark; verify license and serving |
| D-10 | Embedding model | BGE-M3 and multilingual-e5 candidates | Compare retrieval recall, language support, dimensions, cost and latency |
| D-11 | Authentication | Managed identity provider or maintained auth library | Compare pricing, user export/deletion, India availability and lock-in |
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
