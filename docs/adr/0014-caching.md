# 0014. Exact result cache in Valkey; no semantic cache

- Status: Accepted
- Date: 2026-10-02
- Research: [03](../research/03-query-understanding-guardrails-evals.md)

## Decision
- Exact cache keyed by hash(normalized parsed query + filters + k + `index_version`), storing page
  ids only (URLs are signed per request). TTL bounded by the next sync; bumping `index_version`
  after a sync invalidates everything.
- Embedding cache keyed by normalized query text + `embedding_version`.
- No semantic (similarity) cache: embeddings barely separate "ejercicio 3" from "ejercicio 4", so
  it would return the wrong exercise.

## Consequences
Savings are mostly latency and reranker calls (est. 30–40% hit rate in exam weeks, to validate
with logs). No columnar DB needed at ~10k queries/day.
