# 0009. Job orchestration: Procrastinate

- Status: Accepted
- Date: 2026-10-02
- Research: [04](../research/04-ingestion-sync-infra.md)

## Decision
Use Procrastinate (Postgres-backed task queue) for all worker jobs: stages fetch → render → ocr
→ embed → index → link/dedup, provider batch submit + delayed polling, weekly cron.
- One queue per external provider + a Postgres token bucket for rate limits.
- Priorities: admin/owner force-sync > weekly sync > backfill.
- Every job is idempotent, keyed by (content hash, stage, stage_version).

## Alternatives considered
Celery+Redis, Dramatiq (extra broker); arq (maintenance-only); Temporal, Dagster, Prefect (heavy
for 2 devs on one VPS). Hatchet is the upgrade path if we need a UI/built-in rate limiting.

## Consequences
No extra service. Procrastinate's schema SQL is vendored into goose migrations (ADR 0011).
