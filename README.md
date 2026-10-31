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

## Planned documentation

The architecture baseline will be maintained under docs/, covering product requirements, system architecture, AI/model strategy, data model, API contracts, security, deployment, cost assumptions, and implementation roadmap.

## Repository workflow

- main is the protected integration target.
- Work on focused feature or documentation branches.
- Open pull requests for review.
- Do not merge pull requests or deploy without explicit owner approval.
