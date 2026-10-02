# 0011. goose SQL migrations own the schema

- Status: Accepted
- Date: 2026-10-02
- Research: [04](../research/04-ingestion-sync-infra.md)

## Decision
- `db/migrations/*.sql` managed by goose is the single source of truth for the schema.
- Go uses sqlc generated from those migrations; Python uses psycopg3 + SQLAlchemy 2 Core (no ORM
  models that could drift, no Alembic).
- CI applies migrations to a fresh Postgres (with pgvector + BM25 extension) and checks sqlc diff.

## Alternatives considered
Alembic (Python-centric, Go would need to mirror models); Atlas (extra tooling for 2 devs).
