# 04 — Ingestion, Sync & Infrastructure

Status: research, 2026-10-02. Scope: connectors/change detection, format conversion, job orchestration, LangGraph verdict, blob storage, service stacks, sync state model, sizing.

## Context

- Goal: return links to student notes. The canonical `web_url` of every upstream file is a first-class field; everything else is derived.
- Sources: Google Drive (public "anyone with link" or shared with a team identity), OneDrive (public links or Graph), GitHub public repos (md/tex).
- Registry: CEITBA Spring backend exposes users + linked folders/repos (GET/POST/PUT). We pull it; we do not own it.
- Corpus: ~1 TB, ~1.5 M pages; mostly iPad-exported PDFs (handwriting, no text layer), plus images, docx/pptx/xlsx/csv, md, tex.
- Pipeline: mirror → render → VLM OCR → embed → Postgres + pgvector. Weekly cron; upstream deletion ⇒ `stale`; force-sync with per-user cooldown, admin override.
- Constraints: compute-efficient (content-hash gating, batch APIs), team of 2, open source, single VPS with Docker Compose. Python workers (uv), Go query API.

## Connectors & change detection

| Connector | Access mode | Listing / change feed | Change key (skip download if equal) | Content key after fetch | Limits & gotchas |
|---|---|---|---|---|---|
| Google Drive — folder shared with our identity | Service account (users share folder with SA email), read-only scope | `changes.getStartPageToken` once, then `changes.list` with stored token; covers files shared with the identity ([manage changes](https://developers.google.com/workspace/drive/api/guides/manage-changes)). Map changes to registered roots via a cached parent map. Recursive `files.list` (`'<id>' in parents`) for the initial crawl and as a weekly safety net | `sha256Checksum` / `md5Checksum` (binary files); `version` + `modifiedTime` for Google-native Docs/Sheets/Slides (no checksum) | sha256 computed by us | 1 M quota units/min/project, 325 k/min/user; `files.list` = 100 units, `files.get` = 5, download = 200; **1 TB/day egress per project** (initial mirror must span ≥2 days); new 400 M units/day billing threshold with billing details promised later in 2026 ([limits](https://developers.google.com/workspace/drive/api/guides/limits)). 403/429 ⇒ truncated exponential backoff |
| Google Drive — public "anyone with link" folder (not shared to us) | API key or SA; link must carry `resourceKey` if present | No change feed (not in our corpus). Recursive `files.list` per root, diff against `source_files` | Same as above | sha256 | Link-shared items may need the resource key via `X-Goog-Drive-Resource-Keys: <id>/<key>` header ([resource keys](https://developers.google.com/workspace/drive/api/guides/resource-keys)); parse `?resourcekey=` from registered URLs and store it |
| Google-native files (Docs/Slides/Sheets) | Same | Same | `version` | sha256 of export | `files.export` is capped at 10 MB; use the `exportLinks` URLs for larger files ([downloads](https://developers.google.com/drive/api/guides/manage-downloads), [SO](https://stackoverflow.com/questions/60383416/google-drive-api-v2-v3-are-exportlinks-and-file-export-the-same)). Export Docs/Slides → PDF, Sheets → xlsx/csv |
| OneDrive — public share link | **Requires an app + signed-in account.** Entra app registration (multi-tenant + personal accounts) with delegated `Files.Read.All offline_access` for a team Microsoft account | Resolve with `GET /shares/u!{base64url(url)}/driveItem`, then walk `children` recursively, or `delta` when supported | `cTag` (content tag) per file; folder `eTag` changes when descendants change ⇒ prune unchanged subtrees (verify on consumer drives) | `file.hashes.quickXorHash` + our sha256 | Anonymous Graph calls return 401; Microsoft states the Shares API for OneDrive for Business/SharePoint always needs a user context ([shares-get](https://learn.microsoft.com/en-us/graph/api/shares-get?view=graph-rest-1.0)); the legacy anonymous `api.onedrive.com` path broke in late 2024 ([Q&A](https://learn.microsoft.com/en-us/answers/questions/2138092/onedrive-shares-api-behavior-no-longer-reflects-do)). Tenant admins can disable anonymous links (relevant if students use a university M365 tenant) |
| OneDrive — via Graph delta | Same app | `GET /drives/{driveId}/items/{itemId}/delta` → store `@odata.deltaLink`; `?token=latest` to skip backfill ([delta](https://learn.microsoft.com/en-us/onedrive/developer/rest-api/api/driveitem_delta?view=odsp-graph-online)). Delta is the only enumeration Microsoft guarantees complete under concurrent writes | `cTag` | quickXorHash / sha256 | Delta on non-root folders is restricted on Business/SharePoint and may not work for items in *another user's* drive — fall back to recursive `children`. Throttling by resource units per app/tenant; honor `Retry-After` on 429/503 ([throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits), [scan guidance](https://learn.microsoft.com/en-us/onedrive/developer/rest-api/concepts/scan-guidance?view=odsp-graph-online)) |
| GitHub public repo | Anonymous git over HTTPS; optional token for REST | `git ls-remote` for HEAD SHA (no REST quota) → if changed, `git fetch` into a blobless clone (`--filter=blob:none`, [GitHub blog](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)) + sparse-checkout on `*.md`, `*.tex`, `*.pdf`; `git diff --name-status -M <old> <new>` gives A/M/D/R | Git blob SHA per path; repo cursor = commit SHA | sha256 | REST: 60 req/h unauthenticated, 5,000/h with token ([rate limits](https://docs.github.com/en/enterprise-cloud@latest/rest/overview/rate-limits-for-the-rest-api)); recursive Trees API truncates on huge repos — prefer git. Watch for Git LFS pointers |

### Recommendations

- Google identity: **service account**, not OAuth on a team Gmail. An OAuth app left in "Testing" issues refresh tokens that expire after 7 days, and publishing a restricted Drive scope requires Google verification. Users share folders with the SA email (Viewer); public links work with the SA too.
- Microsoft: no supported anonymous crawl. Register one Entra app and keep a team Microsoft account's refresh token in secrets. Re-auth is a manual runbook step.
- Two-level gating: (1) cheap upstream key (`md5/sha256Checksum`, `version`, `cTag`, git blob SHA) decides whether to download; (2) our sha256 over bytes decides whether anything downstream runs. Identical files across users dedupe via sha256.

### Uniform Connector interface (Python)

```python
from dataclasses import dataclass
from datetime import datetime
from typing import BinaryIO, Iterator, Protocol

@dataclass(frozen=True)
class RemoteEntry:
    remote_id: str            # Drive fileId, Graph driveItem id, git path
    path: str                 # display path relative to source root
    mime: str
    size: int | None
    change_key: str           # md5/sha256Checksum | version | cTag | git blob sha
    modified_at: datetime | None
    web_url: str              # link returned to users
    deleted: bool = False
    export_as: str | None = None  # e.g. "application/pdf" for Google Docs

@dataclass(frozen=True)
class ListResult:
    entries: Iterator[RemoteEntry]
    next_cursor: str | None   # changes pageToken | deltaLink | commit sha
    full_snapshot: bool       # True => entries absent from snapshot are deleted

class Connector(Protocol):
    kind: str  # "gdrive" | "onedrive" | "github"
    def resolve(self, url: str) -> dict: ...          # root ids, resourceKey, driveId
    def list(self, source: "Source", cursor: str | None) -> ListResult: ...
    def fetch(self, source: "Source", entry: RemoteEntry, out: BinaryIO) -> None: ...
```

`full_snapshot` lets crawl-based connectors (public Drive, OneDrive children walk) mark missing entries stale, while feed-based ones (changes.list, delta, git diff) emit explicit deletions.

## Format conversion & licensing

| Input | Approach | Library (license) | Notes |
|---|---|---|---|
| PDF → page images | Render at 150–200 DPI, ephemeral | **pypdfium2** (Apache-2.0 / BSD-3; PDFium permissive) | Speed close to PyMuPDF ([pypdfium2](https://redirect.github.com/pypdfium2-team)) |
| PDF (avoid) | — | PyMuPDF (AGPL-3.0 or Artifex commercial, [example](https://github.com/ThalesGroup/fred/issues/1939)) | AGPL forces the whole linking service under AGPL terms; avoid to keep the repo license open (MIT/Apache) |
| PDF text layer | Detect born-digital pages and skip VLM OCR when text is good | pdftext (built on pypdfium2, [repo](https://github.com/datalab-to/pdftext)) or pypdfium2 `get_textpage` | Most iPad exports have no text layer, so this is a small win |
| docx / pptx / odt | Convert → PDF → same render path (keeps equations, diagrams, layout) | LibreOffice headless (MPL-2.0) via **unoserver** daemon container ([unoserver-docker](https://github.com/unoconv/unoserver-docker)) | One conversion uses one core ([Stirling notes](https://docs.stirlingpdf.com/Configuration/Operations/LibreOffice-Parallel-Processing/)); run N daemons = N cores, hard timeout, kill on hang. Separate process = no license coupling |
| xlsx / csv | Direct extraction to markdown tables, no OCR | openpyxl (MIT), stdlib csv | Rendering spreadsheets to pages produces poor OCR input |
| Images (jpg/png/webp) | Normalize (EXIF rotate, max side ~2000 px) | Pillow (MIT-CMU) | |
| HEIC/HEIF | Decode only | **pi-heif** (decoder-only, permissive wheels, [PyPI](https://www.pypi.org/project/pi-heif/1.2.0)) | pillow-heif wheels bundle the x265 encoder (GPL) via plugin; we only need decode |
| md | Chunk source directly | — | No render/OCR |
| tex | Chunk raw source (math stays as LaTeX, which embeds fine) | — | Compiling LaTeX adds a large toolchain for little gain |

License summary: everything linked into our code is Apache/BSD/MIT/MPL; LibreOffice runs as a separate process. The repo can stay under a permissive license.

## Job orchestration

Requirements: idempotent stages (fetch, render, ocr, embed, index), retries with backoff, external API rate limits, provider Batch API submit + poll, priorities (force-sync), cooldown, observability.

| Option | Extra moving parts | Priority | Retry/backoff | Rate limit | Delayed jobs (batch polling) | Dedupe | Notes |
|---|---|---|---|---|---|---|---|
| **Procrastinate** | None (Postgres) | Yes, per job | `RetryStrategy` (exponential) | No (DIY) | `schedule_in` / `schedule_at` | `queueing_lock`, `lock` | Periodic cron tasks, heartbeats + stalled-job retry, task middleware, transactional defer on a caller's connection ([reference](https://procrastinate.readthedocs.io/en/stable/reference.html), [repo](https://github.com/procrastinate-org/procrastinate)) |
| pgmq | Postgres extension | No | Visibility timeout only | No | Delay on send | No | SQS-like primitive; we'd build the worker framework ([PyPI](https://pypi.org/project/pgmq/)) |
| Celery + Redis/Valkey | Broker + result backend (+Flower) | Partial (broker-dependent) | Yes | Per-worker `rate_limit` | ETA/countdown | No | Mature but heavy; state split between broker and Postgres |
| arq | Redis | No | Yes | No | Yes | Job id | Maintenance-only mode since late 2025 ([#510](https://github.com/python-arq/arq/issues/510)) — reject |
| Dramatiq | Redis/RabbitMQ | Yes | Yes | Rate limiter middleware (Redis) | Yes | No | Fine library, still needs a broker |
| Hatchet | Engine + API + dashboard (Postgres-backed; `hatchet-lite` image) | Yes | Yes | **Built-in** ([docs](https://docs.hatchet.run/home/rate-limits)) | Yes | Concurrency keys | Best feature fit + UI; one more service and gRPC dependency |
| Temporal | Server (+ DB, UI) | Task queues | Yes | Via worker options | Durable timers | Workflow id | Overkill for ETL on one VPS |
| Prefect | Server + DB + workers | Work pool priority | Yes | Concurrency limits | Yes | Cache keys | Flow-centric; extra server |
| Dagster | Webserver + daemon + DB | Run queue | Yes | Concurrency pools | Sensors | Asset materialization | Strong for asset lineage, heavy for 2 people |

**Pick: Procrastinate.** It adds no service, keeps job state in the same Postgres as sync state (so enqueuing can be transactional), and covers priority, dedupe, delays and cron. Gaps are small:

- Rate limiting: dedicated queues per external API (`gdrive`, `graph`, `ocr_api`) run by workers with fixed concurrency, plus a Postgres token-bucket row (`SELECT … FOR UPDATE`) for hard per-minute caps. 429 ⇒ raise a retryable error with the `Retry-After` delay.
- Batch API: `ocr_submit` groups pending pages into a provider batch, writes `ocr_batches(batch_id, status)`, and defers `ocr_poll` with `schedule_in={"minutes": 10}`. Poll re-defers itself until the batch reaches a terminal state, then upserts results and defers `embed`. Both are idempotent on `batch_id`.
- Priorities: weekly = 0, user force = 50, admin = 100.
- Observability: OTel spans/metrics via task middleware; jobs table queried directly (Grafana Postgres datasource) for queue depth, failures, age.
- Upgrade path if DIY rate limiting or the missing UI becomes painful: Hatchet (still Postgres-backed).

## LangGraph verdict

**Don't adopt.** Ingestion is deterministic ETL (a job queue fits better than an agent graph). The query path is a fixed retrieve → (rerank) → respond pipeline in Go, and LangGraph/LangChain ship no Go SDK. Adding them would mean a Python hop on the hot path and a large dependency surface for nothing. LangChain document loaders also wrap the libraries above with less control over licensing and errors. The only defensible future use is an isolated Python service for multi-hop/agentic answering or offline eval notebooks. Revisit only if we need that.

## Blob storage

MinIO Community Edition is not viable:
- Admin features were stripped from the community console (May 2025). Binaries and Docker images stopped in Oct 2025, and Docker images were source-only after that ([charts.min.io](https://charts.min.io/), [itsfoss](https://feed.itsfoss.com/link/24361/17326189/minio-moves-away-from-open-source)).
- "Maintenance mode" began on 2025-12-03 ([cloudrumble](https://cloudrumble.net/blog/2025/12/22/minio-to-garage-migration/)). The repo was marked no longer maintained on 2026-02-12 and then archived ([pigsty](https://silo.pigsty.io/blog/post/minio-resurrect/), [pinggy](https://pinggy.io/blog/minio_archived_self_hosted_s3_alternatives/)).
- The `minio/minio` and `minio/mc` Docker Hub repos were deleted on 2026-09-11 ([vonng](https://blog.vonng.com/en/db/silo-is-coming/)).
- It was AGPL-3.0 throughout.

| Option | License | Single-node fit | Maturity | Notes |
|---|---|---|---|---|
| **Garage** | AGPL-3.0 (unmodified service use doesn't affect our repo) | Good: single binary, low RAM, `replication_factor=1` | v2.0 June 2025; NLnet grant 2026 for reliability/perf ([blog](https://garagehq.deuxfleurs.fr/blog/)) | No versioning/Object Lock/SSE ([comparison](https://dev.to/ethan-carter/garage-vs-rustfs-running-s3-on-a-single-node-3d70)), none of which we need. Project committed to never relicensing |
| RustFS | Apache-2.0 | Good: single binary, MinIO-like | 1.0 GA on 2026-09-16, two weeks old ([announcement](https://rustfs.com/blog/announcing-rustfs-1-0-0-ga/)) | Feature-rich; maturity risk. Watch |
| SeaweedFS | Apache-2.0 ([repo](https://github.com/seaweedfs/seaweedfs/)) | OK: `weed server -s3` | Mature | Master/volume/filer model is more to understand; enterprise edition exists |
| Ceph RGW | LGPL | Poor | Mature | Overkill for one node |
| Backblaze B2 | Managed | n/a | Mature | ≈ $6.95/TB-month, egress free up to 3× stored ([swarmify](https://swarmify.com/blog/video-hosting-egress-pricing-compared/)) |
| Cloudflare R2 | Managed | n/a | Mature | $0.015/GB-month (≈ $15/TB), zero egress ([R2 pricing](https://codex-container-api-docs.previews.developers.cloudflare.com/r2/pricing)) |

**Pick: Garage on the VPS**, accessed only through the S3 API (boto3 / aws-sdk-go-v2, path-style, endpoint via env). Workers stay local to the data, with no egress or request fees. Use **B2 as offsite backup** for Postgres dumps, and optionally mirror the blob bucket (re-fetching 1 TB from Drive costs ≥1 day of egress quota). If VPS disk turns out expensive, swap the primary to B2/R2 by changing only the endpoint. Keys: `blobs/sha256/ab/cd/<sha256>` (content-addressed, so it dedupes for free).

## Service stacks & migrations

Go query API:
- Router: std `net/http` (Go 1.22+ method/path patterns) is enough. chi only if middleware ergonomics matter; both are net/http-compatible.
- OpenAPI-first with **oapi-codegen** v2, strict-server mode on std http ([repo](https://github.com/oapi-codegen/oapi-codegen)). The spec lives in `api/openapi.yaml` and is shared with the Spring team.
- DB: **pgx v5 + sqlc** (pgvector via `pgvector-go`). sqlc reads the schema from the migration SQL files.
- Cache/rate limit: **valkey-go**, a rueidis fork with auto-pipelining and client-side caching ([README](https://github.com/valkey-io/valkey-go/blob/main/README.md)).
- Observability: OpenTelemetry (`otelhttp`, `otelpgx`), `log/slog` JSON, Prometheus exporter or OTLP to a collector.

Python workers:
- uv workspace, Python 3.12+.
- **psycopg 3 + SQLAlchemy 2 Core** (no ORM sessions). Bulk upserts (`INSERT … ON CONFLICT`) and `COPY` for vectors matter more than models. Procrastinate uses psycopg 3 natively.
- No Alembic.

Migrations — a single owner: **goose with plain SQL files** in `db/migrations/`, run by a one-shot `migrate` service in Compose before api/workers start.
- Language-neutral; sqlc parses goose migrations directly.
- Alembic would make Python the schema owner and give sqlc nothing to read.
- Atlas is attractive (declarative diffing; the Community Edition is Apache-2.0, [atlasgo](https://atlasgo.io/community-edition)), but its linting moved to paid tiers in v0.38 ([post](https://migrationpilot.hashnode.dev/atlas-migration-linter-alternatives)), and declarative tooling is more than we need.
- Procrastinate's own schema: vendor its SQL migrations into goose files, pinned to the library version.
- CI: run migrations against an ephemeral Postgres, then `sqlc vet` and a Python smoke test that reflects tables used by Core queries.

## Sync state model

```sql
-- Registry mirror (pulled from Spring backend)
CREATE TABLE sources (
  id                 bigserial PRIMARY KEY,
  external_id        text NOT NULL UNIQUE,          -- id in Spring registry
  owner_user_id      text NOT NULL,                 -- Spring user id
  kind               text NOT NULL CHECK (kind IN ('gdrive','onedrive','github')),
  url                text NOT NULL,
  root_ref           jsonb NOT NULL,                -- folderId+resourceKey | driveId+itemId | owner/repo@branch
  cursor             text,                          -- changes pageToken | deltaLink | commit sha
  status             text NOT NULL DEFAULT 'active' CHECK (status IN ('active','disabled','error')),
  last_success_at    timestamptz,
  last_force_sync_at timestamptz,
  created_at         timestamptz NOT NULL DEFAULT now(),
  updated_at         timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sync_runs (
  id           bigserial PRIMARY KEY,
  source_id    bigint NOT NULL REFERENCES sources(id),
  trigger      text NOT NULL CHECK (trigger IN ('cron','user','admin')),
  requested_by text,
  status       text NOT NULL DEFAULT 'queued'
               CHECK (status IN ('queued','listing','processing','succeeded','failed','cancelled')),
  stats        jsonb NOT NULL DEFAULT '{}',         -- seen/added/changed/stale/errors
  error        text,
  created_at   timestamptz NOT NULL DEFAULT now(),
  started_at   timestamptz,
  finished_at  timestamptz
);
-- at most one in-flight run per source
CREATE UNIQUE INDEX sync_runs_one_active ON sync_runs(source_id)
  WHERE status IN ('queued','listing','processing');

-- Content-addressed blobs (dedupe across sources/users)
CREATE TABLE blobs (
  sha256        bytea PRIMARY KEY,
  size          bigint NOT NULL,
  mime          text NOT NULL,
  storage_key   text NOT NULL,
  page_count    int,
  stage         text NOT NULL DEFAULT 'fetched'
                CHECK (stage IN ('fetched','rendered','ocred','embedded','indexed','failed')),
  pipeline_ver  jsonb NOT NULL DEFAULT '{}',        -- {"render":"pdfium-1@200dpi","ocr":"model-x@v3","embed":"model-y@1024"}
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE source_files (
  id            bigserial PRIMARY KEY,
  source_id     bigint NOT NULL REFERENCES sources(id),
  remote_id     text NOT NULL,
  path          text NOT NULL,
  web_url       text NOT NULL,
  mime          text,
  size          bigint,
  etag          text,          -- Graph eTag/cTag, Drive version
  md5           text,          -- Drive md5Checksum
  remote_sha256 text,          -- Drive sha256Checksum / git blob sha
  modified_at   timestamptz,
  sha256        bytea REFERENCES blobs(sha256),     -- null until fetched
  status        text NOT NULL DEFAULT 'active' CHECK (status IN ('active','stale','deleted')),
  last_seen_run bigint REFERENCES sync_runs(id),
  stale_since   timestamptz,
  UNIQUE (source_id, remote_id)
);
CREATE INDEX ON source_files(sha256);
```

Pages, chunks and embeddings key off `blobs.sha256` (not `source_files`). A file shared by many students is OCR'd once, and search results join back to `source_files` (active only) for links.

Idempotency per stage:

| Stage | Idempotency key | Skip condition |
|---|---|---|
| list | `(source_id, sync_run_id)` | — |
| fetch | `(source_file_id, change_key)` (`queueing_lock`) | change_key unchanged; after download, sha256 already in `blobs` ⇒ just relink |
| render | `(sha256, render_ver)` | pages exist for that version |
| ocr | `(sha256, page_no, ocr_ver)` | text exists for that version |
| embed | `(chunk_hash, embed_ver)` | vector exists |
| index | upsert by `(sha256, chunk_no)` | — |

Bumping a `*_ver` in config reprocesses only that stage and the ones after it.

Deletion semantics: an entry is missing from a full snapshot, or a feed reports it deleted ⇒ `status='stale'`, `stale_since=now()`, excluded from results (or shown flagged; open question). It reappears ⇒ `active`. Stale for longer than N weeks ⇒ `deleted`; a GC job removes blobs/vectors with no remaining `active|stale` references.

Weekly cron: a Procrastinate periodic task (`0 3 * * 0`, America/Argentina/Buenos_Aires) refreshes `sources` from the Spring registry, then enqueues one `sync_source` job per active source at priority 0. Sources are spread over the night window to stay under Drive's per-minute/per-day quotas.

Force-sync API contract (Go API; identity/role from a Spring-issued JWT):

```
POST /v1/sources/{id}/sync            -> 202 {sync_run_id, status:"queued"}
  errors: 403 not owner | 409 run already in flight | 429 cooldown (Retry-After, next_allowed_at)
POST /v1/admin/sync  {source_ids?: [...]} -> 202 {sync_run_ids:[...]}   (admin; no cooldown; empty = all)
GET  /v1/sync-runs/{id}               -> {status, stats, started_at, finished_at, error}
GET  /v1/sources/{id}                 -> {status, last_success_at, next_force_allowed_at}
```

Cooldown is enforced atomically in one transaction:

```sql
UPDATE sources SET last_force_sync_at = now()
WHERE id = $1 AND owner_user_id = $2
  AND (last_force_sync_at IS NULL OR last_force_sync_at < now() - $3::interval)
RETURNING id;   -- 0 rows => 429 (or 403 if not owner)
INSERT INTO sync_runs(source_id, trigger, requested_by) VALUES ($1, 'user', $2);  -- unique index => 409
```

Go only writes `sync_runs`; a one-minute Procrastinate periodic task (or `LISTEN/NOTIFY`) defers jobs for queued runs at priority 50 or 100. This keeps Go independent of Procrastinate internals. The cooldown interval (e.g. 7 days) comes from config.

## Infra sizing

Assumptions: OCR and embeddings run through external APIs (no GPU on the VPS); 1024-dim embeddings stored as `halfvec`; ~1–2 chunks per page ⇒ 1.5–3 M vectors; rendered pages are ephemeral (rendered at OCR time, not stored).

| Component | Disk | RAM | CPU |
|---|---|---|---|
| Blob mirror (Garage) | 1 TB + 30% growth ≈ 1.3 TB | ~0.5 GB | low |
| Postgres: vectors | 1.5 M × 2 KB (halfvec 1024) ≈ 3 GB heap; HNSW (m=16) ≈ 3–6 GB; ×2 at 3 M vectors | Keep HNSW resident: ~8–12 GB page cache. HNSW builds need large `maintenance_work_mem`; the default 64 MB is far too small ([notes](https://postgresdba.hashnode.dev/scaling-pgvector-memory-quantization-and-index-build-strategies)) | 2 cores |
| Postgres: text, metadata, jobs | OCR text 1.5 M × ~3 KB ≈ 5 GB; total DB with WAL/bloat ≈ 30–50 GB | (incl. above) | |
| Render workers (pypdfium2) + unoserver ×2 | scratch 20–50 GB | 4–6 GB | 4 cores |
| Go API + Valkey | — | 1–2 GB | 1 core |
| Observability (OTel collector, Grafana, optional Prometheus) | 10–20 GB | 1–2 GB | low |

Recommended VPS/dedicated: **8 vCPU, 32 GB RAM, 2 TB NVMe** (or 512 GB NVMe for Postgres + 2 TB HDD/volume for blobs). Minimum viable: 4 vCPU / 16 GB with halfvec and a ≤1.5 M-vector HNSW. The initial backfill is network/API-bound: Drive egress caps at 1 TB/day/project, and VLM OCR throughput is set by provider batch quotas, not local CPU.

## Recommendation

1. Connectors: Drive via **service account** (`changes.list` for shared folders, recursive `files.list` for public links with resourceKey); OneDrive via **Entra app + team account** (`/shares` → delta or children walk, `cTag`); GitHub via **blobless git + `ls-remote`/`diff`**. Two-level hash gating; a uniform `Connector` protocol.
2. Conversion: pypdfium2, LibreOffice/unoserver sidecar, openpyxl, Pillow + pi-heif. No AGPL libraries linked in.
3. Orchestration: **Procrastinate** on the existing Postgres; Hatchet as the escape hatch.
4. No LangGraph/LangChain.
5. Blob: **Garage** via the S3 API, B2 offsite backups; RustFS on watch. MinIO is out.
6. Go: net/http + oapi-codegen (strict) + pgx/sqlc + valkey-go + OTel. Python: uv + psycopg3/SQLAlchemy Core. **goose SQL migrations** own the schema.
7. Sync model as above: content-addressed blobs, an in-flight-run unique index, atomic cooldown, weekly cron.

## Open questions

- Spring registry: will it push changes (webhook) or must we poll? Does it give stable source ids and user roles (admin flag) in a JWT we can verify?
- Do students share from a university Microsoft 365 tenant (admin may block anonymous/external sharing) or from personal OneDrive?
- Should stale files still be searchable with a "removed upstream" badge, and after how long do we purge?
- Which OCR/embedding providers and models (sets batch API semantics, vector dims and halfvec viability)? See other research docs.
- Folders shared with the SA vs public links: should we require sharing with the SA (enables `changes.list` and avoids resourceKey issues)?
- Can delta work on shared folders in another user's consumer OneDrive? Needs a spike with a real link.
- Cooldown value (3 vs 7 days) and whether it applies per user or per source.
- Is the backup budget enough to mirror blobs to B2 (~$7–9/month for 1–1.3 TB), or only Postgres?

## Sources

- Google Drive: [manage changes](https://developers.google.com/workspace/drive/api/guides/manage-changes), [usage limits](https://developers.google.com/workspace/drive/api/guides/limits), [resource keys](https://developers.google.com/workspace/drive/api/guides/resource-keys), [download & export](https://developers.google.com/drive/api/guides/manage-downloads), [export size limit (SO)](https://stackoverflow.com/questions/60383416/google-drive-api-v2-v3-are-exportlinks-and-file-export-the-same)
- Microsoft Graph: [shares-get](https://learn.microsoft.com/en-us/graph/api/shares-get?view=graph-rest-1.0), [driveItem delta](https://learn.microsoft.com/en-us/onedrive/developer/rest-api/api/driveitem_delta?view=odsp-graph-online), [scan guidance](https://learn.microsoft.com/en-us/onedrive/developer/rest-api/concepts/scan-guidance?view=odsp-graph-online), [throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits), [anonymous Shares API breakage](https://learn.microsoft.com/en-us/answers/questions/2138092/onedrive-shares-api-behavior-no-longer-reflects-do), [anonymous 401](https://learn.microsoft.com/en-us/answers/questions/1195727/anonymously-read-and-upload-to-publicly-shared-one)
- GitHub: [REST rate limits](https://docs.github.com/en/enterprise-cloud@latest/rest/overview/rate-limits-for-the-rest-api), [partial clone](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)
- Conversion: [pypdfium2](https://redirect.github.com/pypdfium2-team), [PyMuPDF AGPL note](https://github.com/ThalesGroup/fred/issues/1939), [pdftext](https://github.com/datalab-to/pdftext), [unoserver-docker](https://github.com/unoconv/unoserver-docker), [LibreOffice parallelism](https://docs.stirlingpdf.com/Configuration/Operations/LibreOffice-Parallel-Processing/), [pi-heif](https://www.pypi.org/project/pi-heif/1.2.0)
- Orchestration: [Procrastinate reference](https://procrastinate.readthedocs.io/en/stable/reference.html), [Procrastinate repo](https://github.com/procrastinate-org/procrastinate), [pgmq](https://pypi.org/project/pgmq/), [arq maintenance-only](https://github.com/python-arq/arq/issues/510), [Hatchet rate limits](https://docs.hatchet.run/home/rate-limits), [Hatchet self-host performance](https://docs.hatchet.run/self-hosting/improving-performance)
- Blob storage: [MinIO source-only](https://charts.min.io/), [itsfoss timeline](https://feed.itsfoss.com/link/24361/17326189/minio-moves-away-from-open-source), [maintenance mode](https://cloudrumble.net/blog/2025/12/22/minio-to-garage-migration/), [archived](https://silo.pigsty.io/blog/post/minio-resurrect/), [archive timeline](https://pinggy.io/blog/minio_archived_self_hosted_s3_alternatives/), [Docker Hub deletion](https://blog.vonng.com/en/db/silo-is-coming/), [Garage blog](https://garagehq.deuxfleurs.fr/blog/), [Garage vs RustFS](https://dev.to/ethan-carter/garage-vs-rustfs-running-s3-on-a-single-node-3d70), [RustFS 1.0 GA](https://rustfs.com/blog/announcing-rustfs-1-0-0-ga/), [SeaweedFS](https://github.com/seaweedfs/seaweedfs/), [R2 pricing](https://codex-container-api-docs.previews.developers.cloudflare.com/r2/pricing), [B2 pricing](https://swarmify.com/blog/video-hosting-egress-pricing-compared/)
- Stacks: [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen), [valkey-go](https://github.com/valkey-io/valkey-go/blob/main/README.md), [Atlas CE](https://atlasgo.io/community-edition), [Atlas lint paywall](https://migrationpilot.hashnode.dev/atlas-migration-linter-alternatives), [pgvector memory](https://postgresdba.hashnode.dev/scaling-pgvector-memory-quantization-and-index-build-strategies), [pgvector halfvec](https://supabase.com/blog/pgvector-0-7-0?)

Content from external sources was rephrased for compliance with licensing restrictions.
