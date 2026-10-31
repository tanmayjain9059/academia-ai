# Security and privacy requirements

Status: mandatory design baseline; implementation controls and threat model must be reviewed before public beta.

## 1. Threat model

Protect student documents, questions, notes, generated answers, account data, credentials and usage records against:
- Cross-tenant access and insecure direct object references.
- Malicious or malformed uploads, parser exploits and archive bombs.
- Prompt injection embedded in uploaded content.
- Credential theft, session abuse and unauthorized downloads.
- Queue/job ownership confusion.
- SSRF through document URLs or provider callbacks.
- Cost abuse through upload, OCR, embedding and generation loops.
- Sensitive data leaking through logs, analytics, caches or model providers.

## 2. Authentication and authorization

- Use a maintained identity provider or well-reviewed authentication library.
- Store password hashes only if managing passwords directly; use a modern adaptive password hashing scheme.
- Enforce workspace membership and role checks server-side for every resource.
- Derive tenant scope from the authenticated principal.
- Never trust user_id, workspace_id, storage key or resource ownership supplied only by the client.
- Apply tenant scope to retrieval, caches, background jobs and export generation.
- Consider PostgreSQL row-level security as defense in depth; test connection-pool context reset.

## 3. Upload safety

- Allowlist extensions and detected media types; verify magic bytes/signatures.
- Enforce per-file size, page count, image dimensions, archive entry count, nesting depth and decompressed-size limits.
- Reject encrypted or malformed files unless an explicit safe workflow is implemented.
- Store originals in private object storage with random object keys.
- Never execute macros, embedded scripts, JavaScript, shell commands or arbitrary files.
- Run parsers/converters in isolated, unprivileged workers with CPU, memory, time and filesystem limits.
- Scan where practical; treat scan results as an additional signal, not a substitute for parser isolation.
- Normalize filenames and never use a user filename as a filesystem path.

## 4. Prompt injection and model safety

- Treat every uploaded document, OCR string, filename, table cell, hyperlink and metadata field as untrusted data.
- Delimit retrieved content as evidence, not instructions.
- System policy and application instructions must not be overridable by source documents.
- Do not allow model output to trigger tools, arbitrary network requests, file access or code execution.
- Validate structured outputs and citation IDs before persistence/rendering.
- Keep secrets and cross-tenant context out of prompts.
- Source-only mode must not silently fall back to general knowledge.

## 5. Storage and transport

- TLS for all network traffic.
- Encryption at rest through managed database/object-storage capabilities.
- Private buckets; short-lived signed URLs scoped to one object and operation.
- Separate production, staging and development credentials.
- Least-privilege service identities.
- Secret manager/environment injection; no secrets in Git or client-side bundles.
- Define backup, restore and object lifecycle policies.
- Deletion must cover originals, derived assets, vector rows, cached artifacts and exports according to the retention policy.

## 6. Logging and monitoring

- Log request/job IDs, tenant ID (opaque), actor ID (opaque), operation, latency and safe error category.
- Do not log document contents, answer contents, auth tokens, signed URLs, raw prompts or full filenames by default.
- Redact provider responses where they may contain source material.
- Alert on repeated authorization failures, unusual upload volume, OCR/inference cost spikes and worker crashes.

## 7. Privacy and policy decisions

Before public launch, decide:
- Data hosting region and cross-border processing.
- Retention defaults and user-controlled deletion.
- Whether model providers retain inputs or use them for training; prefer contractual/no-training controls where available.
- Age eligibility and student privacy obligations in launch markets.
- Terms, privacy notice, abuse reporting and incident response.
- Whether shared subject packs are allowed and how licensing/copyright permissions are verified.

Do not collect Aadhaar, PAN, biometrics or unrelated sensitive personal data for the MVP.
