# Session handoff — resume here

Last updated: 2026-10-02. Phase: **design complete, no code yet, nothing committed.**

## Done
- `.gitignore` keeps private data out (`/Probabilidad Y Estadistica/`, `/samples/`, `/data/`, `.env*`).
- Spec Kit v1.0.13 (`kiro-cli` integration): `.kiro/prompts/speckit.*.md`, `.specify/`.
- Constitution v1.0.0: `.specify/memory/constitution.md`.
- Research `docs/research/00–05`, ADRs `docs/adr/0001–0016`, architecture `docs/architecture/README.md`.
- Diagrams (diagram-design, default style via `.diagram-design`): `docs/diagrams/01–06*.html`.
- Skills vendored at pinned SHAs in `.kiro/skills/` (34 folders, each with `UPSTREAM.txt` + LICENSE);
  review notes in `docs/research/00-community-skills.md` § Vendored. `.github/CODEOWNERS` placeholder.

## Pending decisions (owner: team)
| # | Decision | Blocks |
|---|----------|--------|
| 1 | Golden set: ~40 queries on the private sample with expected pages (private, not in repo), later ≥150 | ADR 0004/0005 bake-offs, ADR 0015 |
| 2 | Microsoft Entra app + team Microsoft account for OneDrive | OneDrive connector (ADR 0012) |
| 3 | Confirm sending notes to external APIs (OCR/embeddings/rerank) | ADR 0003 primary vs self-hosted |
| 4 | CEITBA backend: source-registry endpoint shape, forwarded user id/role/IP, network path | Connectors, ADR 0013 |
| 5 | Force-sync cooldown value (default 7 days) and stale→deleted N weeks | ADR 0012 |
| 6 | VPS size (rec. 8 vCPU / 32 GB / 2 TB NVMe) | Deploy |
| 7 | Repo license (MIT vs Apache-2.0); GitHub team for CODEOWNERS | First commit |
| 8 | Whether to remove sample file paths cited as naming examples in research 02/03 and ADR 0006 | Publishing |

## Proposed ADRs awaiting benchmarks
- 0003 OCR: run the protocol in research 01 on the private sample (< $40).
- 0004 embeddings, 0005 reranker/BM25: run after golden set exists.

## Known follow-ups
- Adapt vendored `commit` / `pr-writer` skills (Sentry conventions) to ours; update their `UPSTREAM.txt` local-changes.
- `skill-creator` evals call `claude -p`; not runnable under Kiro as-is.
- Optional Spec Kit extensions to try: `git`, `bug`, `assess` (adopt); pilot `adrkit`, `threatmodel`.
- Spike: OneDrive delta on folders in other users' personal drives.
- Gemini 3.8 Flash promo pricing ends 2026-12-31 (affects OCR escalation cost).

## Next steps (in order)
1. Make the first commit on a branch (review `git status`; sample folder must not appear).
2. Run the OCR benchmark (research 01) → accept/adjust ADR 0003.
3. `/speckit.specify` features, one at a time:
   1. `001-schema-and-storage` — goose migrations, Garage, Postgres extensions, docker-compose.
   2. `002-connectors-ingestion` — Drive, GitHub (OneDrive after decision 2), mirror, render.
   3. `003-processing` — pre-filter, OCR, metadata merge, embed, dedup, exercise linking.
   4. `004-sync` — weekly cron, force-sync API + cooldowns, stale lifecycle.
   5. `005-query-api` — Go service, parser, hybrid search, rerank, cache, guardrails.
   6. `006-evals` — harness, golden/adversarial sets, CI gate.
   Then `/speckit.plan` → `/speckit.tasks` → `/speckit.implement` → `/speckit.converge` per feature.

## Key facts
- CEITBA API spec: `https://ceitba.org.ar/api/docs.json` (subjects: `GET /v1/subjects?plan=…`,
  `/v1/plans/{planId}/subjects`; auth: Bearer JWT or `X-API-Key`). No notes/sources endpoint yet.
- Sample: 33 iPad PDFs, 360 A4 pages, ~0.69 MB/page; text layers garbled → OCR every page; guides
  combine the LaTeX guide with handwritten solutions that paste the exercise statement.
- Cost estimate: backfill ≈ $1.4k–1.9k; recurring ≈ $175–290/month excl. infra (research 05).
