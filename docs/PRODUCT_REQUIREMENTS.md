# Product requirements

Status: draft baseline for owner review. This document defines the intended product, not a claim that features are implemented.

## 1. Product

Academia AI is a cloud-first, multi-user academic document intelligence platform. A student organizes subjects and academic materials, uploads question papers and reference sources, verifies extracted questions, generates source-grounded answers, edits them, and exports printable study material.

It is not a generic PDF chat wrapper. The product's differentiators are source fidelity, question integrity, structured academic extraction, answer provenance, exam-oriented formatting, and reliable document export.

## 2. Primary users and jobs

- Student: organize semester/subject material, extract questions, prepare answers, revise, export.
- Reviewer/educator (later): review shared subject packs or answer quality with explicit permissions.
- Platform operator: monitor jobs, abuse, costs, model quality, and service health without casually accessing private academic content.

MVP assumes an individual student workspace. Collaboration, institution administration, and public content libraries are later capabilities.

## 3. Core workflow

1. Sign up and create a workspace.
2. Create semester and subject records.
3. Upload files and classify each as a question source or reference source.
4. Validate and store the exact original bytes.
5. Extract native text/layout; use OCR only where needed.
6. Present extraction with source page/slide/sheet and a visual preview.
7. Identify candidate questions without changing their original wording.
8. Let the student verify/correct transcription in a separate versioned field.
9. Index reference material with provenance and subject/unit metadata.
10. Generate an answer using selected sources, answer style, and marks/length target.
11. Display answer, source passages, confidence/coverage warnings, and model metadata.
12. Let the student edit, regenerate, approve, or retain answer versions.
13. Export selected questions and answers to a print-ready PDF.

## 4. File support

### MVP input formats
- PDF (born-digital and scanned)
- DOCX
- PPTX
- XLSX
- JPEG, JPG, PNG, TIFF, WEBP, BMP
- TXT, Markdown, CSV

### Compatibility phase
- DOC, PPT, XLS through isolated conversion (e.g. LibreOffice)
- ODT, ODP, ODS, HTML, EPUB
- ZIP with strict archive limits and safe extraction

### Later media
- Audio: MP3, WAV, M4A with transcription and timestamps.
- Video: MP4, MOV with audio transcription and selected-frame extraction.

Do not promise perfect fidelity for every format. Each format has a declared extraction capability and verification UI. Unsupported, encrypted, corrupted, oversized, or ambiguous files fail with actionable errors.

## 5. Source fidelity and question integrity

These are release-blocking requirements:
- Store uploaded originals byte-for-byte and record SHA-256, size, MIME sniff result, owner, and upload timestamp.
- Original file records are immutable. Re-uploading the same hash may be deduplicated only within the same tenant and with safe reference counting.
- Store native extraction/OCR output as immutable processing artifacts with parser/model/version metadata.
- Store user-verified transcription as a separate, versioned record. Never overwrite raw extraction.
- Store question identity, ordering, hierarchy, marks, source document, page/slide/sheet and region separately from question text.
- Answers are separate versioned records. Regeneration cannot modify question content.
- Every answer source citation resolves to a document, version, page/slide/sheet, block and, where available, bounding box.
- Export must clearly distinguish source-image reproduction from verified transcription.

## 6. Answer modes

- Quick: concise revision answer.
- Exam: structured answer aligned to marks and academic level.
- Detailed: explanatory answer with definitions, steps, examples and diagrams where appropriate.
- Source-only: use only selected uploaded sources; explicitly state when sources do not support an answer.
- Combined: uploaded sources first, then clearly distinguish supplemental general knowledge.

Subject profiles can specify terminology, syllabus scope, answer structure, notation and preferred depth. DBMS profile should provide a short baseline/fundamental explanation before the exam-oriented main answer when that is requested. No subject profile may silently invent syllabus requirements.

## 7. Export

- A4 PDF with readable typography, margins, page numbers and stable page breaks.
- Export by subject, unit, selected questions, questions-only, answers-only or complete Q&A.
- Optional table of contents, marks, source references, diagrams and original question crops.
- Preserve question numbering and ordering.
- Generate export asynchronously for large jobs and provide a private, expiring download link.
- Export is a derived artifact; it never mutates originals or answer history.

## 8. Non-functional requirements

- Multi-tenant isolation at every data boundary.
- Durable object storage and recoverable background jobs.
- Asynchronous processing for expensive extraction, OCR, embeddings, inference and exports.
- Observable job state, retries with bounded attempts, idempotency, and dead-letter handling.
- Accessibility, responsive UI, clear progress and actionable errors.
- Data export and account deletion workflows.
- Cost attribution by tenant and operation; configurable quotas and rate limits.
- Provider-independent model and storage interfaces without premature abstraction.
- Regional/privacy requirements and retention periods must be decided before public launch.

## 9. MVP acceptance criteria

- A user can create a subject and upload supported files.
- The exact original file can be downloaded and its hash matches the uploaded bytes.
- Extraction preserves page/slide/sheet provenance.
- Candidate questions can be reviewed and corrected without changing original extraction.
- Answers are generated from selected sources and include usable citations.
- An answer can be edited and versioned independently of the question.
- A print-ready PDF can be generated and downloaded privately.
- Automated tests prove cross-tenant access is denied for documents, questions, sources, answers, jobs and exports.
- Upload limits, invalid signatures, parser failures, model timeouts and export failures are handled safely.
- No feature is marked complete without automated checks and a manual acceptance pass.
