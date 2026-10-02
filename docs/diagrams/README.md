# Diagrams

Rendered with the `diagram-design` skill as self-contained HTML/SVG files (default style guide). Open any file in a browser. The Mermaid blocks in [`docs/architecture/README.md`](../architecture/README.md) are the source of truth. These files are editorial redraws of those blocks.

| File | Type | What it shows | Mermaid source |
|------|------|---------------|----------------|
| [01-system-context.html](01-system-context.html) | Architecture | Student → CEITBA frontend → backend → ceitba-rag; ceitba-rag reads the catalog/registry back and calls file sources and model providers | §1 System context |
| [02-containers.html](02-containers.html) | Architecture | VPS (Docker Compose): query-api, workers, Postgres, Garage, Valkey, unoserver, plus external APIs, sources and the CEITBA backend | §2 Containers |
| [03-ingestion-pipeline.html](03-ingestion-pipeline.html) | Process | 11 numbered ingestion stages, from source registry to near-dup groups, grouped into lanes. Focal stage: VLM OCR | §3 Ingestion pipeline |
| [04-query-flow.html](04-query-flow.html) | Sequence | `POST /v1/search`: rate limit, grammar parse, `opt` LLM fallback, exact cache, `alt` exact-exercise vs hybrid + rerank, top-k response | §4 Query flow |
| [05-sync-lifecycle.html](05-sync-lifecycle.html) | State machine | Source-file states active / reprocess / stale / deleted and their triggers | §5 Sync lifecycle |
| [06-data-model.html](06-data-model.html) | ER (conceptual) | 10 entities in 3 zones (sync & files, catalog, pages) with cardinalities | §6 Data model |

## How to regenerate or update

1. Edit the Mermaid block in `docs/architecture/README.md` first. It stays the source of truth.
2. Ask the agent to redraw it with the `diagram-design` skill, for example: "redraw §4 of docs/architecture/README.md as a sequence diagram into docs/diagrams/04-query-flow.html with diagram-design". Keep the file names above.
3. Use the default style. The project marker [`/.diagram-design`](../../.diagram-design) (`profile: default`) records that choice, so the skill skips the brand-onboarding prompt.
4. Run the skill's check on the output: `python3 .kiro/skills/diagram-design/scripts/self_check.py docs/diagrams/*.html`.

The skill is vendored and pinned in [`.kiro/skills/diagram-design/UPSTREAM.txt`](../../.kiro/skills/diagram-design/UPSTREAM.txt) (source repo and commit). If you bump that pin, re-check these diagrams.
