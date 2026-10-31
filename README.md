# Academia AI

A cloud-first, multi-user academic document intelligence platform.

Academia AI helps students organize academic materials, preserve original questions, extract structured content from documents, generate source-grounded exam answers, review and edit answers, and export printable study materials.

## Project status

**Architecture planning.** Application implementation has not started.

## Product principles

- Preserve uploaded source files byte-for-byte and never silently rewrite original questions.
- Keep extracted text, verified transcriptions, and generated answers as separate records.
- Ground generated answers in the student's selected academic sources and expose provenance.
- Isolate each user's documents, retrieval context, answers, and exports.
- Support multiple academic document formats through a staged ingestion pipeline.
- Keep the application cloud-first and provider-independent.

## Engineering standards

- Ponytail Ultra: smallest complete, correct implementation.
- Karpathy-inspired guidelines: think before coding, simplicity first, surgical changes, and goal-driven verification.
- Inspect existing code and architecture before modifications.
- Validate trust boundaries; preserve security, accessibility, and data integrity.
- Add tests for non-trivial behavior and run relevant checks before reporting success.
- Never merge into main or deploy without explicit approval.

## Architecture documentation

- Product requirements: `docs/PRODUCT_REQUIREMENTS.md`
- System architecture: `docs/SYSTEM_ARCHITECTURE.md`
- AI/model strategy: `docs/AI_MODEL_STRATEGY.md`
- Data model: `docs/DATA_MODEL.md`
- Data integrity, tenant isolation and evaluation: `docs/DATA_INTEGRITY_AND_EVALUATION.md`
- API contracts: `docs/API_SPECIFICATION.md`
- API lifecycle and mutation contracts: `docs/API_LIFECYCLE_CONTRACT.md`
- Security and privacy: `docs/SECURITY_AND_PRIVACY.md`
- Technical research and provisional decisions: `docs/TECHNICAL_DECISIONS.md`
- Deployment and costs: `docs/DEPLOYMENT_AND_COSTS.md`
- Roadmap: `docs/ROADMAP.md`
- Open decisions: `docs/OPEN_DECISIONS.md`

## Repository workflow

- main is the integration target.
- Work on focused feature or documentation branches.
- Open pull requests for review when a coherent change set is ready.
- Do not merge pull requests or deploy without explicit owner approval.
