# 0008. Query parsing: deterministic grammar first, small LLM fallback; Jev deferred

- Status: Accepted
- Date: 2026-10-02
- Research: [03](../research/03-query-understanding-guardrails-evals.md)

## Context
Queries mix references ("ej 3b G2 proba", "1P 2024 2C") and statement text. Jev (TypeSafe AI)
returns typed probabilistic answers cheaply, but launched 2026-09-15, has paused direct signups,
no Go SDK, no Spanish evaluation, is weak on numbers/dates, and cannot emit free text.

## Decision
- Layer 1: deterministic Go grammar (regex + subject alias table from the CEITBA API catalog +
  typo tolerance) producing fields with rule confidence.
- Layer 2 (only if confidence < 0.85): small model with enum-constrained structured output
  (GPT-5 nano or Gemini Flash-Lite, paid tier). Fields accepted at ≥ 0.7. Expected < $0.05 / 1k queries.
- Statement text = query minus extracted references and filler, done deterministically.
- Parsed fields are soft boosts; the parser cannot set k, offset, sort, or any non-enum value.
- Jev: evaluate later for ingest-time page classification and as an abuse signal, behind the same
  interface. Never a security boundary.

## Consequences
Subject catalog sync from `GET /v1/subjects` and `/v1/plans/{planId}/subjects` of the CEITBA API
is a prerequisite. Parser field accuracy is an eval metric.
