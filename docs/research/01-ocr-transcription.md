# 01 — Page OCR / transcription for handwritten Spanish math notes

Status: research, 2026-10-02. Prices are list prices checked on this date; re-verify before committing spend.

## Context

- Corpus: ~1 TB, ~1.5M pages, nearly all Spanish, mostly iPad handwriting exports (GoodNotes/Notability-style) with heavy math. Embedded text layers are garbled, so every page needs a vision model.
- Page types: official LaTeX guide pages, handwritten solutions that include a pasted screenshot of the exercise statement and its number, exam statement photos plus handwritten solutions, theory summaries, blank pages and notebook covers.
- Index granularity is the page. The OCR output feeds embedding and exercise-reference lookup ("Ejercicio 23 de la Guía 2"). It is never shown as a solution.
- Local sample: 33 PDFs, 360 A4 pages (595×842 pt), produced by iOS Quartz PDFContext, about 690 KB per page on average (the full corpus averages about 670 KB per page). Nothing from the sample was sent to any service for this research.

Cost assumptions used throughout:
- 1 A4 page rendered at 150 dpi (1240×1754).
- About 400 prompt and schema tokens.
- About 700 output tokens: 600 of transcription plus about 100 of JSON metadata.
- Reasoning/thinking set to the minimum. Thinking tokens are billed as output.

Image input tokens per page:

| Provider | Image tokens/page | Basis |
|---|---|---|
| Gemini 3.x, `media_resolution=high` | 1,120 | [documented](https://ai.google.dev/gemini-api/docs/media-resolution) |
| Gemini 2.5 | ~1,550 | 6 tiles × 258 |
| OpenAI | ~2,000 | 32 px patch estimate, [unverified](https://community.openai.com/t/gpt-5-mini-image-input-token-calculation-discrepancy-with-official-faq-formula/1344040/3) |
| Claude | ~1,600 | after resize |

## Options compared

Price per 1k pages uses the assumptions above. "Batch" is the asynchronous API at 50% off.

| Option | Type | Handwriting + math evidence | Output | $/1k pages (batch) | Throughput / hardware |
|---|---|---|---|---|---|
| Gemini 3.1 Flash-Lite | API | No direct OCR benchmark found; the Gemini Flash family scores mid-pack on olmOCR-bench (3 Flash 75.3); Gemini 2.5 Flash was the most faithful transcriber on multi-line handwritten math (FERMAT) | Free-form MD+LaTeX; JSON schema enforced | **0.72** (std 1.43) | Batch target ≤24 h; scales horizontally |
| Gemini 3.5 Flash-Lite | API | Same as above; newer | same | 1.10 | same |
| Gemini 3.8 Flash | API | Strongest Flash tier; use for escalation | same | 1.88 until 2026-12-31, then 3.77 | same |
| Gemini 2.5 Flash-Lite | API | Previous generation, weakest of the family | same | 0.24 | same; deprecation risk |
| Mistral OCR 4.1 | OCR API | olmOCR-bench 85.2 (vendor-reported); vendor says it beats OCR 3 on handwriting | MD + LaTeX, bounding boxes, block types, confidence; custom JSON needs Document AI ($5/1k) or a second LLM pass | 2.00 batch per third-party reports, 4.00 std (+~0.21 classifier pass) | Managed; self-host is enterprise-only |
| OpenAI GPT-5.6 Luna | API | No OCR numbers; the previous nano tier scored 35.2 on olmOCR-bench. GPT-5.4 (full) scored 81.0 | MD+LaTeX; Structured Outputs | ~0.66 (image tokens unverified) | Batch 24 h; Tier 4+ queue 1B tokens |
| OpenAI GPT-5.6 Terra | API | GPT-5.x full models score ~79–81 on olmOCR-bench; GPT-5 led a cursive handwriting test | same | ~6.6 | same |
| Claude Haiku 4.5 | API | olmOCR-bench 61.2 | MD+LaTeX; tool/JSON | ~2.75 | Batch 24 h |
| Claude Sonnet 5.5 | API | The Sonnet 4.6 generation scored 73.9 on olmOCR-bench | same | ~6.5 | same |
| Chandra 2 (4B, open weights) | Self-host | olmOCR-bench 85.9 (vendor); vendor claims handwritten equations; strong on European languages incl. Spanish-group in vendor multilingual eval | HTML/MD/JSON with layout blocks, LaTeX equations | ~0.35–0.45 on rented H100 (+ classifier pass) | 2 pages/s on H100 at 96 concurrency; needs ~16–24 GB GPU; check weights license |
| olmOCR 2 (7B) | Self-host | olmOCR-bench 82.4; among the top 3 in a 100-sample cursive test; tested on 4090/L40S/A100/H100 | Markdown + LaTeX, no layout JSON | ~0.20 (project claim, under $200 per 1M pages) | ≥12 GB VRAM; vLLM |
| PaddleOCR-VL 1.5/1.6 (0.9B) | Self-host | OmniDocBench v1.5 94.5 / v1.6 96.3 (printed); mediocre in handwriting tests (cursive test, OmniHandwritingOCR) | MD + LaTeX + layout | ~0.73 on L40S (third-party estimate) | ~3 GB VRAM; ~1.3k pages/h sustained on L40S (estimate) |
| dots.ocr / dots.mocr (1.7–3B) | Self-host | Good printed layout; authors list formulas as a weakness; weak in the cursive test | JSON layout + MD | similar to PaddleOCR-VL | ~4–8 GB VRAM |
| DeepSeek-OCR / OCR-2 (3B MoE) | Self-host | Lowest score in the cursive test (79%); heavy hallucinated insertions on handwritten formulas (CER >100%) | MD | cheapest per GPU-hour | ~8 GB VRAM |
| Qwen3-VL-8B / Qwen3.5-9B | Self-host (Mac or GPU) | Best aggregate on OmniHandwritingOCR (Qwen3-VL-8B); Qwen3.5-9B scores 77.2 on olmOCR-bench | Promptable: MD+LaTeX and custom JSON in one call | ~150–250 pages/h on M4 Max (derived) | 4-bit ~6 GB; MLX on Mac, vLLM on NVIDIA |
| Nanonets-OCR2-3B (open) / OCR-3 (API) | Self-host / API | OCR2-3B was the best specialized OCR model on OmniHandwritingOCR; OCR-3 tops the vendor-run olmOCR leaderboard (87.4) | MD + LaTeX | — | 3B, ~8 GB VRAM |
| Marker / Docling | Pipeline | Their default OCR engines are not built for handwriting. They are orchestration around a VLM and add little here: every page needs the VLM and layouts are simple | — | — | Skip; use `pdftoppm` plus direct model calls |

## Evidence

- olmOCR-bench is English, mostly printed, and close to saturation. It has no handwriting category; "old scans math" is the closest proxy.
  - Chandra 2 85.9, Mistral OCR 4 85.2, olmOCR 2 82.4, GPT-5.4 81.0, Gemini 3 Flash 75.3, Claude Haiku 4.5 61.2. Sources: [Datalab](https://api.datalab.to/blog/chandra-2), [Mistral](https://mistral.ai/news/ocr-4/), [olmOCR README](https://github.com/allenai/olmocr/blob/main/README.md), [Nanonets leaderboard](https://benchmarking.nanonets.com/benchmarks/olmocr) (vendor-run).
  - Use it only to screen out weak models, not to choose between strong ones.
- Handwritten math is the real risk.
  - [OmniHandwritingOCR](https://arxiv.org/html/2608.18586) (Aug 2026, 13 systems):
    - Qwen3-VL-8B had the best aggregate score, and Nanonets-OCR2-3B was the best specialized model.
    - Every model degrades sharply on long multi-line derivations. Qwen3-VL-8B falls from 86.9 to 70.5 F1 between the easy and hard multi-line subsets.
    - Models "fix" writer errors and insert steps that are not on the page. DeepSeek-OCR showed runaway insertions.
    - Frontier APIs were not included.
  - [FERMAT/PINK](https://arxiv.org/abs/2604.22774v1): GPT-4o is penalized for over-correction, and Gemini 2.5 Flash was the most faithful transcriber of multi-line student math.
    - Implication: the prompt must demand literal transcription, and the benchmark must measure hallucination.
  - [AIMultiple cursive test](https://www.research.aimultiple.com/handwriting-recognition/) (100 English samples, semantic similarity):
    - Top group: GPT-5, Gemini 3 Pro, olmOCR-2.
    - Bottom group: Mistral OCR (pre-4), PaddleOCR-VL, DeepSeek-OCR.
- Spanish: there is no public Spanish handwriting-math benchmark. Datalab's multilingual eval reports Spanish as strong but gives no number. Our own sample is the only valid test.
- Mac throughput is derived, not measured. [vllm-mlx](https://github.com/waybarrios/vllm-mlx/blob/main/docs/benchmarks/image.md) ran Qwen3-VL-8B 4-bit on an M4 Max:
  - About 4 s of prefill for a ~2 MP image and about 40 tok/s decode, so roughly 20 s per 700-token page.
  - That is about 180 pages/h single-stream, or perhaps 2–3× with batching.
- Hosted GPU prices: H100 is about $1.7–3.3/h on neoclouds ([gpucloudcost](https://gpucloudcost.com/)); L40S is about $0.96/h ([Spheron](https://www.spheron.network/blog/best-open-source-ocr-vlm-self-host-gpu-cloud-2026/)).

## Subscription-based seeding verdict

**Verdict: don't.** Using a consumer subscription for the backfill would breach the terms, would take months to years under the quotas, and costs more than a paid batch API run, which totals roughly $1–2k.

| Subscription | Terms of service | Quota reality | Time for 1.5M pages |
|---|---|---|---|
| Claude Pro/Max via Claude Code | The Consumer Terms apply. Advertised limits assume "ordinary, individual usage" ([Claude Code legal](https://code.claude.com/docs/en/legal-and-compliance)). Since 2026-06-15, programmatic use (Agent SDK, `claude -p`) draws from a separate monthly credit at API rates: $20 for Pro, $100/$200 for Max ([InfoWorld](https://www.infoworld.com/article/4171274/anthropic-puts-claude-agents-on-a-meter-across-its-subscriptions.html)) | $200 at Haiku standard rates (~$5.5/1k pages) is about 36k pages/month. Interactive driving hits the 5-hour and weekly windows | ~40 months on Max 20x |
| ChatGPT Plus/Pro via Codex CLI | The consumer Terms of Use forbid automatically or programmatically extracting Output ([OpenAI ToU](https://openai.com/policies/terms-of-use/)). Codex is a coding agent, not a batch OCR endpoint | Per-5-hour message allowances plus an unpublished weekly cap ([codexusage](https://www.codexusage.dev/), [morphllm](https://www.morphllm.com/codex-pricing)) | ≥100 days even ignoring the weekly cap; in practice longer |
| Google AI Pro/Ultra via Gemini CLI | Since 2026-06-18, Gemini CLI no longer serves AI Pro, AI Ultra or free individual accounts ([Tembo](https://www.tembo.io/blog/gemini-cli-pricing), [Google notice (JA)](https://developers.google.com/gemini-code-assist/resources/quotas?hl=ja)). The [geminicli.com quota page](https://geminicli.com/docs/resources/quota-and-pricing/) still lists 1,500/day, which looks stale. On the free API tier, Google uses your content to improve its products, which is unacceptable for third-party student notes | At most 1,500 requests/day if it worked at all | ~1,000 days |

For scale: the recommended batch backfill costs about the same as 5–8 months of a $200 Max plan and finishes in days.

## Pre-filtering and single-call extraction

Pre-filter before any model call. Steps 1–3 cost CPU only.

1. **Content-stream signals (no rendering).** Use `pypdf`/`pikepdf` to read each page's content-stream byte size, vector path count, and image XObject count and area.
   - GoodNotes ink is vector paths and pasted statements are image XObjects.
   - A page with only template paths (ruled or grid lines) and no images is a blank candidate.
2. **Render.** Run `pdftoppm -r 150 -gray`, encode as JPEG q≈85 (~150–250 KB), and make a 256 px thumbnail.
   - Blank check: ink-pixel ratio after subtracting the page's own template, using the per-document median page as background, below a threshold.
   - Confirm blank candidates with the step-1 signals. Covers are mostly large images with little ink, so send them at `media_resolution=low` (280 tokens) instead of skipping them.
3. **Dedup.**
   - Exact: SHA-256 of the normalized rendered bitmap.
   - Near-duplicate: 64-bit pHash with Hamming distance ≤6 marks a candidate, then confirm with SSIM ≥0.98 at 512 px. pHash alone can merge different handwritten pages that share a template.
   - Run dedup across the whole corpus. Official guide pages repeat across many students' combined files, so they are the biggest saving.
   - Key the OCR cache by page hash so renamed or re-uploaded files cost nothing during weekly sync.
4. **One model call per surviving page.** Settings: temperature 0, minimal thinking, schema-enforced JSON (Gemini `responseSchema` / OpenAI Structured Outputs).
   - Instruct the model to transcribe literally and never correct, complete or solve. Unreadable regions become `[ilegible]`.

```json
{
  "transcription_md": "string — Markdown, math as $...$ / $$...$$ LaTeX, reading order",
  "page_kind": "statement | solution | theory | cover | blank | mixed",
  "exercise_refs": [{"label": "Ejercicio|Problema|Item", "number": "23", "sub": "b", "source": "printed|handwritten|screenshot"}],
  "doc_hints": {"guide_number": "2|null", "guide_title": "string|null", "exam_type": "parcial|recuperatorio|final|null", "exam_index": "1|2|null", "year": "2024|null", "term": "1C|2C|null", "subject_guess": "string|null"},
  "has_statement_screenshot": true,
  "language": "es|en|mixed",
  "legibility": "high|medium|low",
  "topics": ["≤5 short Spanish keywords"]
}
```

Escalation: pages with `legibility=low`, invalid JSON, or `page_kind=mixed` with no exercise refs should be re-run on a stronger model. Budget about 10% of pages for this.

## Cost model

Backfill: 1.5M pages, before dedup savings, which are likely 10–30% from blanks and repeated guide pages; measure this. Weekly incremental: 2% of 1.5M = 30k pages/week. Treat that as an upper bound, since the hash cache only re-OCRs changed or new pages.

| Option | $/1k pages | Backfill 1.5M | +10% escalation to Gemini 3.8 Flash (batch) | Weekly 30k |
|---|---|---|---|---|
| **A. Gemini 3.1 Flash-Lite batch** | 0.72 | **$1,070** | +$280 (2026 promo) / +$570 (2027) | **$21** |
| A′. Gemini 3.5 Flash-Lite batch | 1.10 | $1,655 | +$280 | $33 |
| **B. Mistral OCR 4.1 batch + Flash-Lite text classifier pass** | 2.00 + 0.21 | **$3,320** (std: $6,320) | n/a | $66 |
| **C. Chandra 2 self-hosted, rented H100 @ ~$2.5–3/h + classifier pass** | ~0.35–0.42 + 0.21 | **~$850–950** (~210 GPU-h) + ops time | n/a | ~$12 GPU + $6 |
| GPT-5.6 Luna batch (quality unknown) | ~0.66 | ~$990 | — | ~$20 |
| Claude Haiku 4.5 batch | ~2.75 | ~$4,125 | — | ~$83 |
| Single M4 Max, Qwen3-VL-8B 4-bit | power only | ~8–14 months 24/7 | — | ~150 h of compute for 30k pages, more than fits in a week |

Other costs:
- Rendering 1.5M pages is CPU-bound, roughly 10–20 h on a multi-core box.
- Uploading about 300 GB of JPEGs is roughly 7 h at 100 Mbps. Run the job next to the MinIO bucket if possible.
- The Gemini 3.8 Flash promo price ends 2026-12-31 ([Gemini pricing](https://ai.google.dev/gemini-api/docs/pricing)), so schedule the backfill and its escalation pass before then.

## Recommendation

- **Primary: Gemini 3.1 Flash-Lite through the Batch API**, one call per page.
  - Settings: `media_resolution=high`, minimal thinking, the JSON schema above.
  - Escalate about 10% of pages to Gemini 3.8 Flash batch.
  - Switch to 3.5 Flash-Lite only if the benchmark shows a clear gain (more than 2 points on exercise-ref recall or recall@5).
  - Total about $1.1–1.4k for the backfill and about $21/week incremental.
  - Use the paid tier only: the paid tier does not use your data to improve products, the free tier does.
- **Fallback: self-hosted Chandra 2**, or olmOCR 2 if Chandra's license doesn't fit.
  - Run on a rented H100 for the backfill, or one owned 24 GB+ NVIDIA GPU for weekly sync.
  - Add a small text-only classifier pass to produce the JSON fields.
  - This removes vendor and privacy dependence at similar cost but takes more ops work.
  - If the team would rather not run GPUs, Mistral OCR 4.1 batch is the managed fallback at about 3× the cost.
- **Do not use** consumer subscriptions, Apple Silicon for the backfill (too slow; acceptable for development and benchmarks), DeepSeek-OCR (hallucination on handwritten formulas), or Marker/Docling as the OCR layer.

## Benchmark protocol (360-page private sample)

The team runs this; the sample stays private. External APIs are used only with the team's approval and only on paid, no-training tiers. Local models run first.

1. **Labels.**
   - For all 360 pages, record `page_kind`, the visible exercise numbers, and the guide/exam hints. This takes about 2–3 h for one dev.
   - Build a gold subset of 60 pages, stratified across guide-statement, handwritten solution, exam photo, theory, and cover/blank. Transcribe these fully in MD+LaTeX: start from one model's output, then correct by hand against the image, with a second dev spot-checking 15 pages for anchoring bias.
2. **Retrieval set.** Write about 40 realistic queries, half exercise references ("Ej 23 Guía 2") and half topic or formula queries ("integral de x² sen x"), each with known target page(s).
3. **Candidates.**
   - API: Gemini 3.1 Flash-Lite, 3.5 Flash-Lite, 3.8 Flash (ceiling), GPT-5.6 Luna, Mistral OCR 4.1.
   - Local: Chandra 2, olmOCR 2, PaddleOCR-VL 1.6, Qwen3-VL-8B / Qwen3.5-9B on MLX (which also gives a measured Mac throughput).
   - Render at 150 dpi. Also try Gemini at `medium` vs `high` resolution and 200 dpi for the top two.
   - Use the same prompt and schema for every model that can take one. OCR-only models get the classifier pass.
4. **Metrics.**
   - Gold pages: 1-NED/CER on text; normalized LaTeX token F1 on math.
   - Hallucination/over-correction rate: manual review of 30 pages, counting content not visible on the page and any "solved" or corrected steps.
   - All 360 pages: exercise-ref precision and recall, `page_kind` accuracy and confusion matrix (especially blank and cover false positives), and JSON validity.
   - Retrieval recall@5: embed each candidate's output with the same embedding model.
   - Operational: actual tokens and $/page from usage metadata, wall-clock pages/h, and run-to-run stability over two runs at temperature 0.
5. **Decision rule.** Pick the cheapest candidate that meets all of:
   - within 2 points of the best on exercise-ref recall and recall@5;
   - JSON validity ≥99%;
   - hallucination ≤2% of pages.

   Expected API spend is under $40.
6. **Pre-filter check.** Report the blank/cover/dedup hit rates and precision on the 360 pages. Any discarded page that had content is a bug.

## Open questions

- Consent and data governance: may student notes be sent to Google, Mistral or OpenAI at all? If not, the self-hosted fallback becomes the primary.
- Can the official guide PDFs be sourced directly? That would save repeated OCR and give clean statements.
- What "2% changes/week" means: new pages vs rewritten files. This decides whether incremental cost is $21/week or near zero.
- Gemini Batch enqueued-token quota for the team's billing tier, which caps how much of the backfill can run in parallel.
- Mistral OCR 4.1 batch price: $2/1k is reported only by third parties; the official page lists $4.
- OpenAI image token count for Luna: measure it.
- Chandra 2 weights license terms for this project.
- The Gemini 2.5 Flash-Lite deprecation date, in case the benchmark makes it tempting.

## Sources

- [Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing) · [Gemini media resolution](https://ai.google.dev/gemini-api/docs/media-resolution) · [Gemini Batch API](https://ai.google.dev/gemini-api/docs/batch-api)
- [Mistral OCR 4](https://mistral.ai/news/ocr-4/) · [Mistral OCR 3](https://mistral.ai/news/mistral-ocr-3/) · [Mistral pricing](https://docs.mistral.ai/inference/pricing) · [OCR 4 batch price (implicator.ai)](https://www.implicator.ai/mistral-ocr-4-ships-bounding-boxes-for-170-language-document-ai/)
- [GPT-5.6 Luna model page](https://developers.openai.com/api/docs/models/gpt-5.6-luna) · [GPT-5.6 price cut (eesel)](https://www.eesel.ai/blog/gpt-5-6-pricing) · [OpenAI image token discussion](https://community.openai.com/t/gpt-5-mini-image-input-token-calculation-discrepancy-with-official-faq-formula/1344040/3)
- [Claude pricing and batch](https://platform.claude.com/docs/en/about-claude/pricing) · [Claude Code legal & compliance](https://code.claude.com/docs/en/legal-and-compliance) · [InfoWorld on Claude programmatic credits](https://www.infoworld.com/article/4171274/anthropic-puts-claude-agents-on-a-meter-across-its-subscriptions.html)
- [OpenAI Terms of Use](https://openai.com/policies/terms-of-use/) · [Codex usage](https://www.codexusage.dev/) · [Codex limits (morphllm)](https://www.morphllm.com/codex-pricing)
- [Gemini CLI quotas](https://geminicli.com/docs/resources/quota-and-pricing/) · [Gemini CLI consumer shutdown (Tembo)](https://www.tembo.io/blog/gemini-cli-pricing) · [Code Assist quotas notice](https://developers.google.com/gemini-code-assist/resources/quotas?hl=ja)
- [olmOCR README](https://github.com/allenai/olmocr/blob/main/README.md) · [olmOCR 2 paper](https://arxiv.org/html/2510.19817v1) · [Chandra 2](https://api.datalab.to/blog/chandra-2) · [Nanonets olmOCR leaderboard](https://benchmarking.nanonets.com/benchmarks/olmocr) · [dots.mocr](https://github.com/rednote-hilab/dots.mocr)
- [PaddleOCR-VL-1.5 paper](https://arxiv.org/abs/2601.21957) · [DeepSeek-OCR paper](https://arxiv.org/pdf/2510.18234) · [Open-source OCR self-host comparison (Spheron)](https://www.spheron.network/blog/best-open-source-ocr-vlm-self-host-gpu-cloud-2026/)
- [OmniHandwritingOCR](https://arxiv.org/html/2608.18586) · [FERMAT / over-correction](https://arxiv.org/abs/2604.22774v1) · [AIMultiple handwriting benchmark](https://www.research.aimultiple.com/handwriting-recognition/)
- [vllm-mlx image benchmarks](https://github.com/waybarrios/vllm-mlx/blob/main/docs/benchmarks/image.md) · [H100 rental prices](https://gpucloudcost.com/)

Content from sources was rephrased for compliance with licensing restrictions.
