# ceitba-rag

Retrieval engine over ITBA student notes. Given a Spanish query — "Ejercicio 23 de la Guía 2 de
Probabilidad" or the literal statement — it returns links to existing notes that already solve it.
It **never** generates answers.

> Status: design phase. Research and ADRs are proposed; no code yet.
> **Resuming work? Start at [docs/HANDOFF.md](docs/HANDOFF.md).**

## Docs

- [Diagrams](docs/diagrams/README.md) — rendered architecture, pipeline, query, sync, data model
- [Architecture](docs/architecture/README.md) — context, containers, pipelines, data model, taxonomy
- [ADRs](docs/adr/README.md) — decisions and their status
- [Research](docs/research/) — evidence behind each ADR; [cost summary](docs/research/05-cost-summary.md)
- [Constitution](.specify/memory/constitution.md) — non-negotiable project principles

## Workflow (Spec-Driven Development)

Uses [GitHub Spec Kit](https://github.com/github/spec-kit) v1.0.13 with Kiro CLI:

```sh
uv tool install specify-cli --from "git+https://github.com/github/spec-kit.git@v1.0.13"
```

In the agent chat: `/speckit.specify` → `/speckit.plan` → `/speckit.tasks` → `/speckit.implement`
→ `/speckit.converge`. Specs live in `specs/`.

## Privacy

Student notes are consented for indexing, not publication. Never commit notes, samples, OCR output,
or real queries. See `.gitignore`.
