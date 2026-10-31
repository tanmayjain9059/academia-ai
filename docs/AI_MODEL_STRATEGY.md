# AI and document intelligence strategy

Status: candidate strategy. No model is approved for production until evaluated on a representative, versioned benchmark corpus.

## 1. Separate tasks

Do not use one multimodal LLM for every stage. Keep task boundaries explicit:
- Native document parsing.
- OCR and layout analysis.
- Question segmentation/classification.
- Reference chunking and metadata extraction.
- Embedding.
- Retrieval and reranking.
- Answer generation.
- Citation and output validation.

## 2. Extraction routing

1. Detect file type using extension, MIME and file signature; do not trust extension alone.
2. Use Docling as the leading candidate for common document normalization. Its current format matrix includes PDF, modern Office, legacy Office through LibreOffice, OpenDocument, EPUB, markup/text formats, common raster images, and optional audio/video paths. Format support must be validated for feature-level fidelity; it does not guarantee preservation of every note, formula, chart, merged cell or coordinate.
3. Retain specialist native parsers where they preserve more faithful structure or provenance (especially spreadsheet cell/formula metadata and presentation-specific content).
4. Render pages/slides only when visual layout or OCR is needed.
5. Run OCR on scanned pages and images, not indiscriminately on born-digital text.
6. Measure OCR confidence and layout completeness.
7. Escalate only difficult pages (tables, multi-column layouts, formulas, mixed scripts) to heavier document-understanding/VLM models.
8. Keep source image, raw extracted text and normalized blocks for review.
9. Never silently treat low-confidence OCR as verified question text.

Candidate tools/models to benchmark:
- Docling for common document conversion and normalized layout representation (MIT codebase license; individual model/dependency licenses remain separate).
- pypdf and other PDF parsers; review PyMuPDF licensing before adoption.
- python-docx, python-pptx and openpyxl for feature-specific native extraction where needed.
- PaddleOCR PP-OCRv6 as a current printed-text candidate; PP-StructureV3 and PaddleOCR-VL as layout/document escalation candidates. The PaddleOCR repository is Apache-2.0, but review individual model weights and transitive dependencies.
- LibreOffice in a restricted worker for legacy Office conversion.
- Pillow/OpenCV for image preprocessing.

## 3. Question extraction

Use deterministic parsing first for numbering, headings, marks and common structures. Use a classifier/LLM only for ambiguous segmentation and classification.
- Preserve raw text exactly as extracted.
- Preserve source block IDs, coordinates and page order.
- Represent parent/child structure (e.g. Q3, Q3(a), Q3(a)(i)).
- Store marks and unit labels as extracted metadata with confidence.
- User corrections create a new verified transcription version.
- Do not let a model paraphrase the question during extraction.

## 4. Retrieval

Initial design: PostgreSQL full-text search + pgvector.
- Chunk along headings, paragraphs, slides, table rows and other structural boundaries.
- Store workspace_id, subject_id, document_id, document version, source type, unit, page/slide/sheet and block IDs.
- Apply access and subject filters before vector ranking.
- Combine lexical and vector candidates; deduplicate by source block.
- Evaluate reranking only if it improves answer evidence retrieval enough to justify latency and cost.
- Citations must point to exact source locations, not merely a document title.

Candidate embedding models: BGE-M3 and smaller multilingual alternatives such as multilingual-e5 families. Select based on language coverage, retrieval recall, latency, hosting cost and license—not popularity.

## 5. Answer generation

Benchmark open-weight instruction models from Qwen and other suitable families using a stable inference API. Candidate sizes should include a compact model and a stronger model; do not hard-code a model name until evaluation.

Generation inputs:
- Original/verified question text and question identity.
- Marks and answer mode.
- Subject, unit and syllabus constraints.
- Retrieved evidence passages with source IDs.
- User instructions and output format.
- Explicit instruction to distinguish source-supported content from supplemental general knowledge.

Output contract:
- Structured answer sections.
- Claims linked to source IDs where supported.
- Missing-evidence or low-confidence warning.
- Model ID/version and prompt-template version.
- Token usage and latency metadata.

Never claim citations support text unless the referenced passage exists and was included in the request. A validator can check citation IDs and coverage mechanically; semantic correctness still requires evaluation and user review.

## 6. Model serving

Keep an inference adapter with a small contract for:
- chat/generation,
- embeddings,
- reranking,
- OCR/VLM where applicable,
- health and model metadata.

Hugging Face is a useful model discovery and prototype ecosystem, but do not equate free Space compute with unlimited production inference. Current HF docs distinguish CPU Basic, paid upgraded GPU hardware and usage-based Inference Providers. Free-user Inference Provider credits are currently listed as $0.10/month and are subject to change. See:
- https://huggingface.co/docs/hub/en/spaces-overview
- https://huggingface.co/docs/hub/en/spaces-gpus
- https://huggingface.co/docs/inference-providers/pricing

The production adapter must support a replaceable GPU provider or self-hosted serving stack. Evaluate vLLM, Hugging Face Inference Endpoints and other GPU hosts using measured throughput, batching, cold starts, data handling, availability and cost.

## 7. Evaluation before selection

Build a versioned evaluation set containing:
- Born-digital and scanned PDFs.
- Low-resolution and skewed scans.
- English and relevant Indian-language content.
- Multi-column layouts, tables, formulas and diagrams.
- DOCX/PPTX/XLSX with tables, images, notes and merged cells.
- Question papers with nested numbering and marks.
- Reference notes with conflicting or incomplete statements.
- Handwriting as a separately scored category.

Measure:
- OCR character/word error rate and exact question transcription rate.
- Layout/table extraction accuracy.
- Question boundary and hierarchy F1.
- Retrieval recall@k and citation-source accuracy.
- Answer factuality, syllabus compliance and citation support.
- Latency, throughput, memory, failure rate and cost per page/answer.

Use human-reviewed ground truth. Do not select models based only on leaderboard rank or a handful of hand-picked examples.
