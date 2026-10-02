# 0001. Retrieval-only engine; never generate answers

- Status: Accepted
- Date: 2026-10-02

## Context
Students ask things like "necesito resolver la integral de x² sen x" or "Ejercicio 23 de la Guía 2
(93.24)". Many students have already solved these in their own notes. Generating solutions would
add liability (wrong answers), cost, and a large prompt-injection surface.

## Decision
- The engine only parses the query and returns ranked references to existing pages.
- Default k = 5; callers may request more up to a server-side cap (ADR 0013).
- Each result: original source URL (Drive/OneDrive/GitHub) first, mirrored copy as fallback,
  page number, metadata (subject, doc type, guide/exercise, year/term), duplicate-group size,
  and `last_synced_at`.
- Every response carries a disclaimer (Spanish) that results may be inaccurate or outdated
  relative to the owner's source and that CEITBA does not vouch for content correctness.
- No generative model on the query path ever receives retrieved note content. The reranker
  (ADR 0005) receives stored excerpts but returns only scores.

## Alternatives considered
- RAG answer generation — rejected: liability, cost, injection surface; out of product scope.
- Snippet generation by LLM — rejected for the same reasons; snippets, if any, are verbatim
  stored transcription excerpts.

## Consequences
- Quality is purely a ranking problem, measurable with recall@k/MRR (ADR 0015).
- Injection impact on the query path is bounded to filter manipulation (ADR 0013).
