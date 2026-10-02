# 0007. Near-duplicate grouping with a canonical page

- Status: Accepted
- Date: 2026-10-02
- Research: [02](../research/02-retrieval-embeddings-rerank.md)

## Decision
- Signals: exact sha256 (file and page image), 64-bit perceptual hash (pgvector `bit` Hamming
  HNSW), MinHash on transcription. Embedding cosine is a tie-breaker only — two different
  students solving the same exercise are similar but not duplicates.
- Union-find into `dup_groups`; one canonical page per group (earliest/most complete source).
- Results show the canonical page with "N posibles duplicados" and their source links.
- Dedup runs before OCR (saves cost) and after OCR (catches re-exports).

## Consequences
Only canonical pages are in the HNSW partial index, shrinking it and preventing result floods.
