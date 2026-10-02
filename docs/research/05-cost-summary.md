# Cost summary

Consolidates estimates from research 01–04 (prices checked 2026-10-02). All figures USD, to be
re-validated after the benchmark on the private sample. Assumptions: ~1.5M pages, ~2% weekly
change, 2k daily students × ~5 queries ≈ 10k queries/day ≈ 300k/month.

## One-time backfill

| Item | Estimate | Source |
|------|----------|--------|
| OCR — Gemini 3.1 Flash-Lite Batch | ~$1,070 | [01](01-ocr-transcription.md) |
| OCR escalation (~10% pages, larger model) | $280–570 | [01](01-ocr-transcription.md) |
| Embeddings (~1.05B tokens) | $10–210 (model-dependent) | [02](02-retrieval-embeddings-rerank.md) |
| Exercise-link LLM adjudication (ambiguous band) | < $50 (estimate) | [02](02-retrieval-embeddings-rerank.md) |
| **Total** | **≈ $1.4k–1.9k** | |

Pre-OCR dedup (repeated official guide pages) should lower OCR cost; not yet measured.

## Recurring (monthly)

| Item | Lean | Default | Source |
|------|------|---------|--------|
| Weekly incremental OCR + embed (~30k pages/week) | ~$90 | ~$90 | [01](01-ocr-transcription.md) |
| Query embedding | ~$1 | ~$1 | [02](02-retrieval-embeddings-rerank.md) |
| Reranking (60% of queries after cache/exact matches) | ~$80 (`rerank-3-lite`) | ~$195 (`rerank-3`) | [02](02-retrieval-embeddings-rerank.md) |
| Parser LLM fallback (10–30% of queries) | < $5 | < $5 | [03](03-query-understanding-guardrails-evals.md) |
| VPS 8 vCPU / 32 GB / 2 TB NVMe | provider-dependent | provider-dependent | [04](04-ingestion-sync-infra.md) |
| Offsite backup (B2, ~1 TB) | provider-dependent | provider-dependent | [04](04-ingestion-sync-infra.md) |
| **Total excl. infra** | **≈ $175** | **≈ $290** | |

Per query (default): ≈ $0.001. Reranking dominates; the cost lever is reranking fewer queries
(exact exercise matches, cache hits) or using the lite reranker.

For VPS and storage prices use the provider's pricing calculator.

## Not recommended: subscription-based seeding
Consumer LLM subscriptions are capped or contractually unsuitable for ~1.5M programmatic calls
(details in [01](01-ocr-transcription.md)); the paid batch backfill costs roughly 5–8 months of a
$200/month plan and completes in days.
