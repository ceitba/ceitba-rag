# 0004. Embeddings: Qwen3-Embedding-8B @1024 halfvec

- Status: Proposed — confirmed by bake-off on the golden set
- Date: 2026-10-02
- Research: [02](../research/02-retrieval-embeddings-rerank.md)

## Context
Spanish, math-heavy transcriptions; users type plain-text math. No public Spanish-math retrieval
benchmark exists. Embedding the whole corpus once costs ~$10–210 with any top candidate, so the
choice is reversible.

## Decision
- Default: Qwen3-Embedding-8B via hosted API, Matryoshka-truncated to 1024 dims, stored as
  `halfvec(1024)`. Open weights (Apache-2.0) avoid provider lock-in.
- Bake-off candidates: Gemini Embedding 2, Voyage 4 (large for docs / lite for queries).
- Embed a normalized `plain_text` (LaTeX linearized to words, e.g. `\int x^2 \sin x` → "integral x
  al cuadrado seno x") plus metadata header; the same normalizer is applied to queries.
- Page-image embeddings are an optional later signal, gated by an A/B test.
- Model id + dims + normalizer version form `embedding_version`; changing it re-embeds only.

## Alternatives considered
OpenAI text-embedding-3, Cohere embed v4, jina v4 (non-commercial weights), bge-m3 self-host,
ColPali/ColQwen (≈380 GB of patch vectors; not indexable in pgvector).

## Consequences
HNSW ≈4 GB RAM for 1.5M vectors; VPS ≥32 GB RAM.
