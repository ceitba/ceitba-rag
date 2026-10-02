# 0015. Golden set + Go eval harness as merge gate

- Status: Accepted
- Date: 2026-10-02
- Research: [03](../research/03-query-understanding-guardrails-evals.md)

## Decision
- Golden set (Spanish, JSONL): ≥150 queries with graded relevant page ids, including ≥30 pairs
  that ask for the same exercise by reference and by statement text. Stored privately (not in the
  public repo), referenced by version.
- Adversarial set (~80): query injection, poisoned pages, enumeration replays.
- Metrics: recall@5, MRR@5, nDCG@10, exercise-link accuracy, pair consistency, parser field accuracy.
- Harness: `go test` against a frozen index snapshot with committed query embeddings (near-zero
  cost per PR). promptfoo nightly for the fallback parser prompt and red-teaming.
- Gates: recall@5 and nDCG@10 may not drop > 2 pp; exercise-link accuracy ≥ 95%; adversarial
  invariants 100%.

## Alternatives considered
Ragas/DeepEval — focused on generated-answer quality, which we do not produce.
