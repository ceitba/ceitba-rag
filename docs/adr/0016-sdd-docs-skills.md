# 0016. Spec Kit SDD, diagram-design, vendored community skills

- Status: Accepted
- Date: 2026-10-02
- Research: [00](../research/00-community-skills.md)

## Decision
- GitHub Spec Kit v1.0.13 (installed from the official git tag) with the `kiro-cli` integration;
  prompts in `.kiro/prompts/speckit.*.md`, constitution in `.specify/memory/constitution.md`,
  feature specs in `specs/NNN-name/`. Brownfield changes use `/speckit.specify` against existing
  specs; `/speckit.converge` closes each feature.
- Diagrams: vendored `cathrynlavery/diagram-design` skill (`.kiro/skills/diagram-design`, pinned
  commit in `UPSTREAM.txt`). Outputs (HTML/SVG) in `docs/diagrams/`.
- Community skills from research 00 are vendored at pinned SHAs with `UPSTREAM.txt` + LICENSE,
  reviewed by hand, under CODEOWNERS. No auto-update marketplaces.

## Consequences
CC-BY(-SA) licensed skills keep their license and attribution inside their folders.
