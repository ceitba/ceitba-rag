# ceitba-rag Constitution

## Core Principles

### I. Retrieval Only (NON-NEGOTIABLE)
ceitba-rag finds and ranks existing student notes; it never solves, summarizes, or generates
answers. Every response is a ranked list of references (original source URL first, mirrored
copy as fallback) plus a disclaimer that the index may be inaccurate or out of date with the
owner's source. No generative model on the query path may receive retrieved note content;
only score-only rerankers may.

### II. Contracts First
Components communicate only through versioned contracts: the SQL schema (owned by goose
migrations), the OpenAPI spec of the query/sync API, and the job payload schemas. Contract
changes require a spec update and an ADR when they are breaking. The engine stays portable:
CEITBA's frontend/backend integrate over HTTP; no frontend concerns live here.

### III. Measured Quality (Evals Gate)
Retrieval changes (models, prompts, chunking, ranking, parser rules) merge only with an eval
run against the golden set. Regressions beyond the thresholds in `docs/adr/0015-evals.md` block merge.
Adversarial/abuse invariants must pass at 100%.

### IV. Secure and Abuse-Resistant by Default
Assume hostile queries and hostile note content. Model outputs are structured, enum-constrained,
and never trusted as authority. Enforce rate limits, result caps, no listing endpoints, short-lived
signed URLs, spend circuit breakers, and audit logs. Secrets never enter the repo.

### V. Private Data Stays Private
Indexed notes are consented for indexing, not publication. Raw notes, sample corpora, OCR
output, and real user queries are never committed. Fixtures use synthetic or explicitly
licensed content.

### VI. Incremental and Idempotent Processing
Every pipeline stage (fetch, render, OCR, embed, index) is idempotent, keyed by content hash
and stage version. Only changed content is reprocessed. Upstream deletions are soft (stale)
before hard deletion.

### VII. Simplicity
Fewest moving parts that meet the requirements: one Postgres, one blob store, one cache.
New services, frameworks, or providers require an ADR with the rejected alternatives.

## Technical Constraints

- Query path: Go service (net/http, oapi-codegen, pgx, sqlc). Ingestion/sync: Python (uv,
  psycopg3/SQLAlchemy Core, Procrastinate). Schema: goose SQL migrations.
- Storage: PostgreSQL + pgvector (+ BM25 extension per ADR 0006), S3-compatible blob store
  accessed only through the S3 API, Valkey for cache and rate limits.
- Model providers are behind interfaces so they can be swapped; model ids and stage
  versions are configuration, never hard-coded.
- Language: code, specs, docs, and ADRs in English; prompts, eval queries, and user-facing
  strings in Spanish.

## Development Workflow

- Spec-Driven Development with GitHub Spec Kit: constitution → specify → plan → tasks →
  implement → converge, per feature under `specs/`.
- Architecture decisions are recorded as ADRs in `docs/adr/`; research backing them lives
  in `docs/research/`.
- Diagrams are produced with the vendored `diagram-design` skill and live in `docs/diagrams/`.
- Every PR: builds, linters, and tests pass; evals run when retrieval behavior changes;
  docs/diagrams updated when architecture changes.
- Third-party agent skills are vendored at a pinned commit with `UPSTREAM.txt` and reviewed
  before merge.

## Governance

This constitution supersedes other practices. Amendments require a PR that updates this file,
bumps the version (MAJOR: principle removed/redefined; MINOR: principle/section added;
PATCH: clarifications), and is approved by a maintainer. Reviewers verify compliance on every PR;
deviations must be justified in the PR or an ADR.

**Version**: 1.0.0 | **Ratified**: 2026-10-02 | **Last Amended**: 2026-10-02
