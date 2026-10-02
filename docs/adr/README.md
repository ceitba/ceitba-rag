# Architecture Decision Records

Format: [MADR-lite](0000-template.md). One decision per file, numbered, never renumbered.
Superseded ADRs stay in place with `Status: Superseded by NNNN`.

Status values: `Proposed` (awaiting team review / benchmark), `Accepted`, `Superseded`, `Rejected`.

| ADR | Decision | Status | Research |
|-----|----------|--------|----------|
| [0001](0001-retrieval-only.md) | Retrieval-only engine; never generate answers | Accepted | — |
| [0002](0002-service-split-go-python.md) | Go query API + Python workers; no LangGraph/LangChain | Accepted | [04](../research/04-ingestion-sync-infra.md) |
| [0003](0003-ocr-transcription.md) | VLM page transcription via Gemini Flash-Lite Batch; no subscription seeding | Proposed (benchmark) | [01](../research/01-ocr-transcription.md) |
| [0004](0004-embeddings.md) | Qwen3-Embedding-8B @1024 `halfvec`, behind a provider interface | Proposed (bake-off) | [02](../research/02-retrieval-embeddings-rerank.md) |
| [0005](0005-hybrid-search-rerank.md) | Hybrid search in Postgres (HNSW + BM25, RRF) + API reranker | Proposed (bake-off) | [02](../research/02-retrieval-embeddings-rerank.md) |
| [0006](0006-exercise-linking.md) | Canonical `exercises` table linking guides and solutions | Accepted | [02](../research/02-retrieval-embeddings-rerank.md) |
| [0007](0007-near-duplicates.md) | Near-duplicate grouping with canonical page | Accepted | [02](../research/02-retrieval-embeddings-rerank.md) |
| [0008](0008-query-parsing.md) | Deterministic grammar first, small LLM fallback; Jev deferred | Accepted | [03](../research/03-query-understanding-guardrails-evals.md) |
| [0009](0009-job-orchestration.md) | Procrastinate (Postgres-backed jobs) | Accepted | [04](../research/04-ingestion-sync-infra.md) |
| [0010](0010-blob-storage.md) | Garage (S3 API) instead of MinIO | Accepted | [04](../research/04-ingestion-sync-infra.md) |
| [0011](0011-schema-migrations.md) | goose SQL migrations own the schema; sqlc on Go | Accepted | [04](../research/04-ingestion-sync-infra.md) |
| [0012](0012-sync-change-detection.md) | Per-connector change detection, soft deletes, force-sync cooldowns | Accepted | [04](../research/04-ingestion-sync-infra.md) |
| [0013](0013-guardrails-abuse.md) | Guardrails and anti-scraping controls | Accepted | [03](../research/03-query-understanding-guardrails-evals.md) |
| [0014](0014-caching.md) | Exact result cache in Valkey; no semantic cache | Accepted | [03](../research/03-query-understanding-guardrails-evals.md) |
| [0015](0015-evals.md) | Golden set + Go eval harness as merge gate | Accepted | [03](../research/03-query-understanding-guardrails-evals.md) |
| [0016](0016-sdd-docs-skills.md) | Spec Kit SDD, diagram-design, vendored community skills | Accepted | [00](../research/00-community-skills.md) |
