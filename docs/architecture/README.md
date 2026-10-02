# ceitba-rag architecture

Retrieval engine that maps a Spanish natural-language query ("Ej 23 Guía 2 de Proba" or a pasted
statement) to a ranked list of existing student-note pages. It never generates answers (ADR 0001).

Mermaid blocks below are the editable source of truth (they render on GitHub). Polished versions
are produced with the `diagram-design` skill into `docs/diagrams/` (see [diagrams](#diagrams)).

## 1. System context

```mermaid
flowchart LR
  student([Student]) --> fe[CEITBA frontend / app]
  fe --> be[CEITBA backend<br/>Spring · auth · source registry]
  be -- "query, force-sync<br/>API key + user id" --> rag[[ceitba-rag]]
  rag -- "subjects catalog,<br/>sources registry" --> be
  rag -- "mirror files" --> src[(Google Drive · OneDrive · GitHub)]
  rag -- "OCR · embed · rerank<br/>(batch / API)" --> llm[(Model providers)]
```

## 2. Containers

```mermaid
flowchart TB
  subgraph vps[VPS · Docker Compose]
    api[query-api · Go<br/>parse · search · rerank · sign · force-sync]
    workers[workers · Python<br/>Procrastinate jobs]
    pg[(Postgres<br/>pgvector · BM25 · jobs)]
    s3[(Garage · S3 API<br/>blobs · OCR JSON)]
    vk[(Valkey<br/>cache · rate limits)]
    lo[unoserver<br/>LibreOffice]
  end
  be[CEITBA backend] --> api
  api --> pg & vk & s3
  api --> prov[(Embedding / rerank / parser APIs)]
  workers --> pg & s3 & lo
  workers --> ext[(Drive · OneDrive · GitHub)]
  workers --> ocr[(OCR batch API)]
  workers --> be
```

| Container | Language | Responsibility | ADR |
|-----------|----------|----------------|-----|
| query-api | Go | Query endpoint, result redirect/presign, force-sync requests, rate limits, cache | 0002, 0013, 0014 |
| workers | Python | Connectors, render, OCR, embed, index, dedup, exercise linking, cron | 0002, 0009, 0012 |
| Postgres | — | Metadata, transcriptions, vectors, BM25, job queue, sync state | 0005, 0009, 0011 |
| Garage | — | Content-addressed file mirror, raw OCR JSON | 0010 |
| Valkey | — | Exact result cache, embedding cache, token buckets | 0013, 0014 |

## 3. Ingestion pipeline

Each stage is an idempotent job keyed by `(sha256, stage, stage_version)`; unchanged content is
skipped at every hop (ADR 0012).

```mermaid
flowchart LR
  reg[Source registry<br/>CEITBA API] --> list[list_changes<br/>per connector]
  list -->|changed upstream key| fetch[fetch → sha256]
  fetch -->|new sha256| store[(Garage)]
  fetch --> render[render pages<br/>pypdfium2 · LibreOffice · openpyxl]
  render --> pre[pre-filter<br/>blank/cover · pHash dedup]
  pre --> ocr[VLM OCR batch<br/>JSON per page]
  ocr --> cls[metadata merge<br/>path rules ⊕ OCR hints]
  cls --> emb[embed plain_text]
  emb --> idx[(pages · page_embeddings · BM25)]
  cls --> link[exercise linking]
  idx --> dedup[near-dup groups]
  link --> idx
```

Non-PDF formats: docx/pptx → PDF via LibreOffice then the same path; xlsx/csv parsed directly;
md/tex chunked from source (no OCR); images (incl. HEIC) treated as single pages.

## 4. Query flow

```mermaid
sequenceDiagram
  autonumber
  participant BE as CEITBA backend
  participant API as query-api (Go)
  participant VK as Valkey
  participant LLM as Parser LLM (fallback)
  participant PG as Postgres
  participant RR as Reranker API
  BE->>API: POST /v1/search {q, filters, k} + user id
  API->>VK: rate limit check
  API->>API: grammar parse (subject, guide, ex, exam, year/term) + statement text
  opt rule confidence < 0.85
    API->>LLM: enum-only structured output
  end
  API->>VK: exact cache lookup (parsed query + index_version)
  alt exact exercise match
    API->>PG: exercises → page_exercise_links
  else
    API->>PG: HNSW (halfvec) ∪ BM25, filters, RRF, collapse dup groups
    API->>RR: rerank top-50 (page ids → stored excerpts)
  end
  API->>VK: store page ids
  API-->>BE: top-k {source_url, mirror_token, page, metadata, dup_count, last_synced_at, disclaimer}
```

The reranker receives stored transcription excerpts but returns only scores; no model output on
the query path is ever shown to the user or used as instructions.

## 5. Sync lifecycle

```mermaid
stateDiagram-v2
  [*] --> active: first seen
  active --> active: upstream key unchanged (skip)
  active --> reprocess: upstream changed & sha256 changed
  reprocess --> active: stages done
  active --> stale: missing upstream
  stale --> active: reappears
  stale --> deleted: N weeks
  deleted --> [*]: GC blobs no longer referenced
```

Triggers: weekly cron (Sun 03:00 ART); owner force-sync (own sources, cooldown default 7 days);
admin force-sync (any/all). Responses `202`, `409` (run in flight), `429` + `Retry-After`.

## 6. Data model (conceptual)

```mermaid
erDiagram
  SOURCE ||--o{ SOURCE_FILE : contains
  SOURCE ||--o{ SYNC_RUN : has
  BLOB ||--o{ SOURCE_FILE : "content of"
  SOURCE_FILE ||--o{ PAGE : "rendered into"
  PAGE ||--|| PAGE_EMBEDDING : has
  PAGE }o--o| DUP_GROUP : "member of"
  PAGE ||--o{ PAGE_EXERCISE_LINK : ""
  EXERCISE ||--o{ PAGE_EXERCISE_LINK : ""
  SUBJECT ||--o{ EXERCISE : ""
  SUBJECT ||--o{ PAGE : ""
```

Full DDL sketches: [research 02](../research/02-retrieval-embeddings-rerank.md) (retrieval tables),
[research 04](../research/04-ingestion-sync-infra.md) (sync tables). The authoritative schema will
be `db/migrations/` (ADR 0011).

## 7. Metadata taxonomy

Values are enums in the schema; Spanish aliases live in the parser's alias table.

| Field | Values / format | Primary signal | Fallback |
|-------|-----------------|----------------|----------|
| `subject_code` | CEITBA catalog id, e.g. `93.24` | folder name ↔ catalog alias match | OCR header ("Probabilidad y Estadística 93.24") |
| `doc_type` | `guide`, `guide_solution`, `theory_summary`, `class_notes`, `midterm` (parcial), `retake` (recuperatorio), `final`, `lab`, `tp`, `other` | folder/filename rules (`Practica`, `Resumenes`, `Finales`, `1P`, `1R`, `1F`) | OCR `page_kind` + model |
| `exam_instance` | `1P`, `2P`, `1R`, `2R`, `1F`, `2F`… | filename | OCR header |
| `year`, `term` | `2024`, `1C`/`2C` | filename | OCR header |
| `guide_no` | integer | filename (`G2`, `guia2`, `Guía 2`) | OCR header |
| `exercise_labels` | `23`, `3b`, `3.b`… per page | OCR `exercise_refs` | inherited from previous page |
| `page_kind` | `statement`, `solution`, `theory`, `cover`, `blank`, `mixed` | OCR | — |
| `owner`, `source`, `source_url`, `page_no` | from connector | connector | — |
| `metadata_confidence` | 0–1 per field + `method` | — | — |

Rule: path/filename metadata beat model output; disagreements lower confidence and are
quarantined for review (ADR 0013).

## 8. Repository layout (planned)

```
api/openapi.yaml           # query-api contract
db/migrations/             # goose SQL (schema owner)
services/query-api/        # Go
workers/                   # Python (uv): connectors, pipeline, jobs
evals/                     # harness + adversarial suite (golden set stored privately)
deploy/                    # docker-compose, Garage/Valkey/Postgres config
docs/{adr,research,architecture,diagrams}/
specs/                     # Spec Kit features
```

## Diagrams

Rendered versions of sections 1–6 (self-contained HTML/SVG, `diagram-design`). To regenerate them, see
[docs/diagrams/README.md](../diagrams/README.md).

- [01 System context](../diagrams/01-system-context.html)
- [02 Containers](../diagrams/02-containers.html)
- [03 Ingestion pipeline](../diagrams/03-ingestion-pipeline.html)
- [04 Query flow](../diagrams/04-query-flow.html)
- [05 Sync lifecycle](../diagrams/05-sync-lifecycle.html)
- [06 Data model](../diagrams/06-data-model.html)
