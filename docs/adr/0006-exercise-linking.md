# 0006. Canonical exercises table linking guides and solutions

- Status: Accepted
- Date: 2026-10-02
- Research: [02](../research/02-retrieval-embeddings-rerank.md)

## Context
"Ejercicio 23 de la Guía 2" and the literal statement text must return the same solution pages.
Folder conventions vary (`subject/practice/guia1.pdf`, `G2 - Calculo De Probabilidades.pdf`), but
official guides/exams are usually present and solution pages often embed a screenshot of the
statement with its number.

## Decision
- `exercises` table: one row per (subject_code, source_kind {guide, exam}, guide_no | exam id,
  exercise label, year/term), with statement transcription and statement embedding. Built from
  pages classified as `statement` in official guide/exam documents.
- `page_exercise_links(page_id, exercise_id, method, confidence)`. Methods, in order:
  1. explicit label on page + statement similarity check;
  2. statement similarity alone above a margin;
  3. LLM adjudication in the ambiguous band (batch, ingest-time only).
  Pages without a visible header inherit the previous page's exercise within the same document.
- Query: parser yields (subject, guide, exercise) → direct lookup; free-text statements are
  matched against `exercises.statement_embedding` first, then fan out to linked solution pages.

## Consequences
Linking accuracy becomes a tracked eval metric (target ≥ 95%). Exercises that change numbering
across years are distinct rows linked by statement similarity.
