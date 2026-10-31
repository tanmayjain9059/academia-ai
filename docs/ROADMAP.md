# Delivery roadmap

Status: proposed sequence. Each phase requires review of acceptance evidence before moving on.

## Phase 0 — Architecture baseline (current)

Deliver:
- Product requirements and explicit MVP boundaries.
- System architecture and trust boundaries.
- Data model and API contracts.
- AI/OCR benchmark plan.
- Security/privacy baseline.
- Deployment and cost strategy.
- Data integrity, tenant isolation and deletion lifecycle specification.
- Open decision register and acceptance criteria.

Exit criteria:
- Owner reviews and approves the architecture.
- Critical unknowns are resolved or explicitly accepted as risks.
- Hosting and model budget ceiling is documented.
- Benchmark corpus design, annotation approach and numeric quality gates are agreed after baseline measurements.
- No application runtime code is written before authorization.

## Phase 1 — Repository and engineering foundation

- Establish protected-branch workflow and CI.
- Choose package managers, runtime versions and formatting/lint/type-check tools.
- Configure environment validation, secrets handling and local development.
- Establish backend/frontend project structure only after stack decisions.
- Add health checks, structured logging and error conventions.
- Add minimal smoke tests.

Exit criteria: reproducible local setup; CI passes; no secrets committed.

## Phase 2 — Identity, workspace and storage

- Authentication and workspace ownership.
- Subject/semester CRUD.
- Private object-storage upload intents and finalization.
- File signature/size validation and quota enforcement.
- Document listing, deletion and audit metadata.
- Database RLS and composite ownership constraints.
- Cross-workspace negative tests across API, SQL, pooled connections, object storage and worker jobs.

Exit criteria: cross-workspace tests deny unauthorized access using the least-privilege runtime role; pooled transactions show no RLS context leakage; original hash round-trips.

## Phase 3 — Document ingestion

- Format router and native parsers.
- PDF/page rendering and OCR adapters.
- Unified block model with page/slide/sheet provenance.
- Background jobs, retries and progress.
- Extraction review UI.

Exit criteria: benchmark corpus passes format-specific quality thresholds; failures are recoverable and observable.

## Phase 4 — Question extraction and integrity

- Question segmentation and numbering hierarchy.
- Marks/unit detection with confidence.
- Candidate review and verified transcription versions.
- Original question preview/crop.
- Regression tests for immutable source and transcription history.

Exit criteria: question text never mutates original file or raw extraction; benchmark precision/recall meets agreed thresholds.

## Phase 5 — Reference indexing and retrieval

- Structural chunking.
- Embeddings and pgvector.
- PostgreSQL lexical search.
- Tenant/subject/unit filtering.
- Citation provenance.
- Retrieval evaluation harness.

Exit criteria: recall@k and tenant isolation meet agreed thresholds.

## Phase 6 — Answer engine

- Inference adapter and model registry/config.
- Answer modes and subject profiles.
- Prompt construction and structured output.
- Citation validation and source coverage warnings.
- Answer versioning, editing and approval.
- Token/latency/cost tracking.

Exit criteria: human-reviewed answer evaluation meets agreed factuality, citation and syllabus-compliance thresholds.

## Phase 7 — PDF export

- A4 templates and typography.
- Question numbering and answer version selection.
- Source references and optional source crops.
- Async generation and private expiring download.
- Visual regression and PDF text/layout tests.

Exit criteria: exports print correctly across representative long/short answers and diagrams.

## Phase 8 — Private beta hardening

- Threat-model review and dependency/license review.
- Load tests and queue saturation behavior.
- Backup/restore test.
- Usage quotas and cost alerts.
- Accessibility and responsive review.
- Privacy notice, terms, retention and deletion workflows.
- Invite-only beta.

Exit criteria: release checklist approved; no unresolved critical/high security issues.

## Phase 9 — Public launch and scale

Only after beta evidence:
- Billing/plans if required.
- Collaboration and shared packs if justified.
- Multi-region strategy if required.
- Dedicated vector infrastructure or GPU serving only if metrics justify it.
- SLOs, on-call, incident and abuse workflows.

Do not build speculative multi-region, Kubernetes, microservices or social features into the MVP.

See [Data integrity, tenant isolation and evaluation specification](DATA_INTEGRITY_AND_EVALUATION.md) for the detailed security, lifecycle and quality gates.
