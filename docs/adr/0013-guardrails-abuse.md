# 0013. Guardrails and anti-scraping controls

- Status: Accepted
- Date: 2026-10-02
- Research: [03](../research/03-query-understanding-guardrails-evals.md)

## Context
Primary concerns: abuse and scraping of the index. Secondary: prompt injection in queries and in
note content (indirect), ranking poisoning, denial of wallet.

## Decision
- query-api reachable only from the CEITBA backend (private network + API key); backend forwards
  user id, role, and client IP.
- Token-bucket rate limits per user and per IP in Valkey; daily per-user quotas.
- k default 5, hard cap (e.g. 20); no deep pagination; no listing/browse endpoints.
- Results carry opaque signed result tokens; mirror access goes through a redirect endpoint that
  issues 60–300 s presigned URLs and is itself rate-limited.
- Enumeration anomaly detection (sequential exercise sweeps, high distinct-subject rate) → throttle/flag.
- Query input length cap; parser output is enum/number-only; no generative model sees retrieved
  content (the reranker sees excerpts but only returns scores).
- Ingest: folder/filename metadata are ground truth; classifier disagreements are quarantined;
  OCR prompt treats page text as data.
- Spend circuit breaker: on budget breach, fall back to deterministic parsing and skip reranking.
- Audit log of queries (user id hashed), sync requests, and mirror downloads.

## Consequences
Mapped to OWASP LLM Top 10 (2026) in research 03. Adversarial eval suite must pass 100% (ADR 0015).
