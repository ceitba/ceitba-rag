# 0002. Go query API + Python workers; no LangGraph/LangChain

- Status: Accepted
- Date: 2026-10-02
- Research: [04](../research/04-ingestion-sync-infra.md)

## Context
The query path is a fixed, non-agentic pipeline: parse → embed → hybrid search → rerank → sign
URLs. With cloud model APIs it is I/O-bound HTTP + SQL. Ingestion needs the Python document
ecosystem (pypdfium2, openpyxl, provider batch SDKs). The CEITBA backend (Spring) will call us over HTTP.

## Decision
- `services/query-api` (Go): stdlib `net/http`, oapi-codegen strict server, pgx + sqlc, valkey-go,
  OpenTelemetry. Owns query, result redirect/sign, and the force-sync request API (writes
  `sync_runs` rows only).
- `workers/` (Python, uv): connectors, render, OCR, embed, index, dedup, linking; Procrastinate jobs.
- Shared contracts: SQL schema (ADR 0011), `api/openapi.yaml`, job payload JSON schemas.
- No LangGraph or LangChain. Ingestion is ETL; the query path is deterministic; LangGraph has no
  Go SDK. Provider SDKs/HTTP clients are wrapped behind small interfaces.
- Service-to-service auth: CEITBA backend → query-api with an API key over a private network;
  the backend forwards the user id and role. No end-user auth is re-implemented here.

## Alternatives considered
- All-Python (FastAPI) — viable, but Go gives a single static binary, lower latency/memory for the
  hot path, and matches team preference.
- LangGraph query flow — adds a framework for a 5-step linear pipeline with no branching agents.

## Consequences
- Two languages to maintain; mitigated by schema/OpenAPI contracts and sqlc.
- Embedding and parser prompts must be callable from both sides (ingest-time and query-time);
  model ids and prompt versions live in shared config.
