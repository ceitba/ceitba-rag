# 0003. VLM page transcription via batch API; no subscription-based seeding

- Status: Proposed — final model chosen by the benchmark protocol in research 01
- Date: 2026-10-02
- Research: [01](../research/01-ocr-transcription.md)

## Context
The sample (33 iPad-exported PDFs, 360 A4 pages, ~0.69 MB/page) shows embedded text layers are
garbled, so every page needs a vision model. Content is handwritten Spanish math plus pasted
screenshots of exercise statements. Extrapolated corpus: ~1.5M pages.

The team proposed seeding "locally" with an LLM subscription it already pays for. Research found:
Claude scripted usage draws from a capped API-rate credit (~40 months for the backfill), ChatGPT
consumer terms forbid programmatic extraction, Gemini CLI no longer serves consumer plans.
Local VLMs on Apple Silicon would take ~8–14 months.

## Decision
- One VLM call per page returning strict JSON: transcription (Markdown + LaTeX), `page_kind`
  {statement, solution, theory, cover, blank, mixed}, `exercise_refs`, guide/exam hints,
  `has_statement_screenshot`, `language`, `legibility`, topics. Prompt (Spanish) demands literal
  transcription — never correcting or solving.
- Primary candidate: Gemini 3.1 Flash-Lite via Batch API (~$1.1k backfill, ~$21/week incremental),
  escalating low-legibility/invalid-JSON pages (~10%) to Gemini 3.8 Flash batch.
- Fallback: self-hosted Chandra 2 / olmOCR 2 on a rented GPU for the backfill if quality or
  policy requires it; Mistral OCR batch as managed alternative.
- Pre-filter before OCR: blank/cover detection, exact + perceptual-hash dedup; cache OCR by page
  image hash and `ocr_version`.
- Store transcriptions in Postgres and the raw JSON in blob storage so re-embedding never re-OCRs.
- Subscription-based seeding is rejected.

## Alternatives considered
See research 01 table (Mistral OCR, GPT/Claude vision, DeepSeek-OCR, Marker/Docling, local Mac).

## Consequences
- Backfill should run before Gemini 3.8 Flash promo pricing ends (2026-12-31) if escalation uses it.
- Benchmark on the private sample is required before acceptance (budget < $40).
- Provider is behind an `OCRProvider` interface; switching re-runs OCR only for new `ocr_version`.
