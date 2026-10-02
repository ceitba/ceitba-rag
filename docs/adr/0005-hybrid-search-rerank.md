# 0005. Hybrid search in Postgres + API reranker

- Status: Proposed — reranker and BM25 extension confirmed by bake-off
- Date: 2026-10-02
- Research: [02](../research/02-retrieval-embeddings-rerank.md)

## Context
Vector search misses literal tokens (exercise numbers, "G2", "1P 2024"); keyword search misses
paraphrases. Metadata filters (subject, doc type, year) are frequent.

## Decision
- pgvector HNSW (m=16) on `page_embeddings.embedding halfvec(1024)`, partial index on canonical
  pages; iterative index scans for filtered queries; filter columns denormalized onto
  `page_embeddings`.
- BM25 via ParadeDB `pg_search` (Spanish stemmer) on `plain_text` plus a math-token field.
  Fallback: native FTS with a `spanish_unaccent` config if the extension is unsuitable.
- Fuse with RRF (k=60) over top-100 each, collapse duplicate groups, rerank top-50.
- Reranker: Voyage `rerank-3` (default) or `rerank-3-lite` (lean); exact exercise matches skip
  reranking.
- Parsed metadata are soft boosts unless the user set them explicitly as filters.

## Alternatives considered
Dedicated vector DB (Qdrant/Weaviate) — extra service, no need at 1.5M vectors.
Cohere Rerank 4 — ~2× cost. Self-hosted reranker — needs a GPU (~$650/month).

## Consequences
- ParadeDB is AGPL; run unmodified as an extension (no repo license impact). Re-evaluate if it
  complicates Postgres upgrades.
- Estimated $100–215/month at 300k queries/month, dominated by reranking (see cost summary).
