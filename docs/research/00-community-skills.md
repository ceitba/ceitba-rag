# 00 — Community agent skills and Spec Kit extensions

Status: research, nothing installed · Date: 2026-10-02

## Context

ceitba-rag is a public repo containing Python ingestion/sync workers, a thin Go query API, Postgres + pgvector, S3-compatible blob storage, VLM OCR, embeddings/rerank, and evals/guardrails. It uses Spec Kit 1.0.13 (latest release is [v1.0.13](https://github.com/github/spec-kit/releases), 2026-09-29) through Kiro CLI. Skills are loaded from `.kiro/skills/<name>/SKILL.md`, and `diagram-design` is already there.

Trust baseline: the skills ecosystem has no curation by default. skills.sh is an open, zero-curation registry ranked by telemetry, and it sits at the center of the ecosystem's security debate ([Ry Walker](https://rywalker.com/research/skills-sh)). Trail of Bits says published skills have been found with backdoors and malicious hooks ([trailofbits/skills-curated](https://github.com/trailofbits/skills-curated)). Spec Kit maintainers check only that community catalog entries are well-formed. They do not audit the code ([Spec Kit community extensions](https://github.github.com/spec-kit/community/extensions.html)). So we only take skills from vendor or org accounts, or from individuals with a long track record, and only the specific subfolders we need.

Notes on the data: "Updated" is the repo's last push date from the GitHub API, checked 2026-10-02. "SHA" is HEAD on that day, recorded only as a reference point. Re-check both at vendoring time.

## Candidates

| Name / skills of interest | URL | Maintainer | License | Updated | What it does | Trust | Rec |
|---|---|---|---|---|---|---|---|
| cc-skills-golang: `golang-error-handling`, `golang-context`, `golang-database`, `golang-testing`, `golang-security`, `golang-observability`, `golang-concurrency`, `golang-lint` | [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) | Samuel Berthe (samber/lo author) | MIT | 2026-10-01 | ~45 Go-only skills (language, testing, security, observability). Dev-workflow skills are out of scope by design. | High: individual, but a well-known Go maintainer; 3.4k★ | **Adopt** (subset) |
| golang-skills | [cxuu/golang-skills](https://github.com/cxuu/golang-skills) | cxuu | Apache-2.0 | 2026-06-20 | Go style distilled from the Google and Uber guides, with progressive-disclosure references | Medium: 165★, slower cadence | Maybe (fallback) |
| go-agent-skills / golang-agent-skill | [eduardo-sl](https://github.com/eduardo-sl/go-agent-skills), [saisudhir14](https://github.com/saisudhir14/golang-agent-skill) | individuals | MIT / ? | 2026 | Go style guidance | Low: low traction | Skip |
| Trail of Bits skills: `modern-python`, `property-based-testing`, `differential-review`, `sharp-edges`, `supply-chain-risk-auditor` | [trailofbits/skills](https://github.com/trailofbits/skills) | Trail of Bits | CC-BY-SA-4.0 | 2026-09-28 | Security audit and testing skills; `modern-python` covers uv/ruff/ty tooling | High: security firm; 7.3k★ | **Adopt** (subset) |
| supabase-postgres-best-practices | [supabase/agent-skills](https://github.com/supabase/agent-skills) | Supabase | MIT | 2026-09-28 | Postgres rules for query performance, indexes, schema, and connections, ranked by impact, with SQL examples | High: vendor | **Adopt** (skip the `supabase` platform skill) |
| JetBrains/skills `postgres-best-practices` | [JetBrains/skills](https://github.com/JetBrains/skills) | JetBrains | none detected | 2026-06-29 | A copy of the Supabase skill | Medium: a mirror | Skip (use upstream) |
| evals (`eval-audit`, `evaluate-rag`, `error-discovery`, `write-judge-prompt`, `validate-evaluator`, `generate-synthetic-data`, `write-code-eval`) | [ai-evals-course/evals-skills](https://github.com/ai-evals-course/evals-skills) | Hamel Husain & Shreya Shankar | Apache-2.0 | 2026-09-24 | Error analysis, LLM-judge calibration, and a RAG eval workflow ([blog](https://hamel.dev/blog/posts/evals-skills/)) | High: recognized eval practitioners | **Adopt** |
| hamelsmu/evals-skills | [hamelsmu/evals-skills](https://github.com/hamelsmu/evals-skills) | Hamel Husain | MIT | archived | Deprecated predecessor of the evals repo above | n/a | Skip (archived) |
| OWASP secure-agent-playbook: `prompt-injection-test`, `llm-risk-assess`, `code-review-security` | [OWASP/secure-agent-playbook](https://github.com/OWASP/secure-agent-playbook) | OWASP | CC-BY-4.0 | 2026-09-25 | OWASP LLM Top 10 risk assessment and prompt-injection testing against a taxonomy | Med-high: official OWASP org, but young (182★) | **Adopt** (subset) |
| Sentry skills: `code-review`, `pr-writer`, `commit`, `find-bugs`, `gha-security-review`, `skill-scanner` | [getsentry/skills](https://github.com/getsentry/skills) | Sentry | Apache-2.0 | 2026-09-29 | Engineering workflow skills (review, PRs, commits, GitHub Actions audit, skill audit) | High: vendor | **Adopt** (subset; adapt commit/PR conventions to ours) |
| Docker skills: `docker-project-foundations`, `docker-build-strategies`, `docker-compose-patterns`, `docker-destructive-guardrails` | [docker/skills](https://github.com/docker/skills) ([docs](https://docs.docker.com/ai/skills/)) | Docker | Apache-2.0 | 2026-10-01 | Official Dockerfile/Compose guidance plus guardrails. The `docker-agent-*` and `sandboxes-*` skills are not relevant. | High: vendor | **Adopt** (subset) |
| Anthropic skills: `skill-creator`, `doc-coauthoring`, `mcp-builder` | [anthropics/skills](https://github.com/anthropics/skills) | Anthropic | Apache-2.0 per skill (docx/pdf/pptx/xlsx are source-available only) | 2026-09-29 | Reference skills. `skill-creator` covers writing and evaluating our own skills. | High: spec author | **Adopt** `skill-creator`; Maybe `doc-coauthoring` |
| addyosmani/agent-skills (`documentation-and-adrs`, `code-review-and-quality`, `security-and-hardening`) | [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) | Addy Osmani | MIT | 2026-10-02 | Broad engineering-discipline set, including its own spec-driven workflow | Med-high: well-known individual; 100k★ | Maybe (its SDD skills overlap with Spec Kit) |
| ADR skill | [vercel/ai `skills/adr-skill`](https://github.com/vercel/ai/blob/main/skills/adr-skill/SKILL.md) / [github/awesome-copilot `create-architectural-decision-record`](https://github.com/github/awesome-copilot/blob/main/skills/create-architectural-decision-record/SKILL.md) | Vercel / GitHub | Apache-2.0 / MIT | 2026 | Create, supersede, and index ADRs | High: vendor, but each sits inside a larger repo | Maybe (or write a 1-page in-house ADR skill) |
| obra/superpowers | [obra/superpowers](https://github.com/obra/superpowers) | Jesse Vincent | MIT | 2026-09-27 | TDD, systematic debugging, planning workflow | High popularity (294k★) | Skip (opinionated workflow that clashes with Spec Kit phases) |
| mattpocock/skills, vercel-labs/agent-skills | [mattpocock](https://github.com/mattpocock/skills), [vercel-labs](https://github.com/vercel-labs/agent-skills) | individual / Vercel | MIT / none detected | 2026 | TypeScript and React/Next focused | High, but off-stack | Skip |
| wshobson/agents, affaan-m/everything-claude-code, sickn33/agentic-awesome-skills | [wshobson](https://github.com/wshobson/agents), [affaan-m](https://github.com/affaan-m/everything-claude-code), [sickn33](https://github.com/sickn33/agentic-awesome-skills) | individuals | MIT | 2026-10 | Mass collections (180+ skills; the sickn33 entries are tagged `risk: unknown`) | Low-med: volume over curation | Skip |
| microsoft/skills | [microsoft/skills](https://github.com/microsoft/skills) | Microsoft | MIT | 2026-10-01 | Azure SDK grounding | High, but off-stack | Skip |
| ⚠ Look-alike forks/mirrors: `aihai/anthropics-skills`, `guchengod/anthropics-skills`, `bkootaz/https-github.com-anthropics-skills`, `VastFuture/sentry-skills`, `diegosouzapw/awesome-omni-skill`, third-party "install" pages (eliteai.tools, typingmind, lobehub) | various | unknown | — | — | Re-hosted copies of official skills | **Low / typosquat-ish** | Skip; vendor only from the canonical org repo |

Discovery sources: [skills.sh](https://www.skills.sh/), [VoltAgent/awesome-agent-skills](https://github.com/voltagent/awesome-agent-skills), [royalpinto007/awesome-agent-skills](https://github.com/royalpinto007/awesome-agent-skills) (a security-aware short list), and [officialskills.sh](https://officialskills.sh/). These are used for discovery only. Never install from them directly.

Portability note: many of these repos ship as Claude Code plugins, with hooks, agents, and commands next to the skills. Kiro only consumes the `SKILL.md` folder and its `references/`/`scripts/`, so the plugin wrappers get dropped. Any bundled `scripts/` must be reviewed line by line before vendoring.

## Spec Kit extensions

Official extensions in the core catalog are maintained in [github/spec-kit](https://github.com/github/spec-kit/tree/main/extensions):

| Extension | What | Rec |
|---|---|---|
| `git` | Feature-branch creation and numbering, validation, remote detection | **Adopt** |
| `bug` | Assess, fix, and validate bug reports, with per-bug reports in `.specify/bugs/<slug>/` | **Adopt** |
| `assess` | Go/kill intake before `/speckit.specify`, stored in `.specify/assessments/` | **Adopt** (useful for "should we add hybrid search / a new OCR model?") |
| `github` | Creates GitHub issues from `tasks.md` | Maybe (once the issue tracker is active) |
| `agent-context` | Manages agent instruction files with plan references | Maybe |
| preset `lean` | Minimal core command prompts | Skip for now |

Community extensions (all MIT/Apache, single maintainers, low stars; none are reviewed by Spec Kit):

| Extension | Repo | Maintainer | License | Updated | What | Rec |
|---|---|---|---|---|---|---|
| adrkit | [mbeacom/adrkit](https://github.com/mbeacom/adrkit) | Mark Beacom | Apache-2.0 | 2026-10-02 (v0.1.4) | Loads governing decisions into context, checks plans against them, drafts ADRs from plans | Maybe: best ADR fit, but pre-1.0 and 18★. Pilot on one feature. |
| threatmodel | [NaviaSamal/spec-kit-threatmodel](https://github.com/NaviaSamal/spec-kit-threatmodel) | NaviaSamal | MIT | 2026-09-10 (v2.1.2) | Read-only OWASP LLM Top 10 (2026) analysis of spec artifacts | Maybe: read-only, so low risk. Pairs with the OWASP skills. |
| red-team | [ashbrener/spec-kit-red-team](https://github.com/ashbrener/spec-kit-red-team) | Ash Brener | MIT | 2026-09-01 | Adversarial pre-plan spec review (prompt injection, silent failures); writes a report, no auto-edits | Maybe |
| security-review | [DyanGalih/security-review](https://github.com/DyanGalih/security-review) | DyanGalih | MIT | 2026-08-23 (v2.0.0) | Secure-by-design audits per plan/task/PR | Maybe (overlaps with Sentry + ToB skills) |
| review | [ismaelJimenez/spec-kit-review](https://github.com/ismaelJimenez/spec-kit-review) | ismaelJimenez | MIT | 2026-04-09 | Post-implementation multi-agent code review (read-only) | Skip (stale; covered by skills) |
| reconcile | [stn1slv/spec-kit-reconcile](https://github.com/stn1slv/spec-kit-reconcile) | Stanislav Deviatov | MIT | 2026-08-25 | Updates spec artifacts to match drifted implementation | Maybe (later) |
| trace | [Quratulain-bilal/spec-kit-trace](https://github.com/Quratulain-bilal/spec-kit-trace) | Quratulain-bilal | MIT | 2026-05-12 | Requirement → test traceability matrix | Skip for now (4★) |

## Recommendation

Adopt these 8 skill sources, taking only the listed subfolders:

1. `samber/cc-skills-golang` @ `8e899e20ff0cd4dc524af3993e4c62d8ee8c5717`: Go API (error-handling, context, database, testing, security, observability, concurrency, lint).
2. `trailofbits/skills` @ `82fe8226252622fa807643bdca1710901198553a`: modern-python, property-based-testing, differential-review, sharp-edges, supply-chain-risk-auditor. CC-BY-SA-4.0 is share-alike, so the vendored files stay under that license with attribution in their own folders.
3. `supabase/agent-skills` @ `544bfc56c89afe2b87b20017a59b2c6e9502a1fb`: supabase-postgres-best-practices. I did not verify whether it covers pgvector (HNSW/IVFFlat tuning). If it doesn't, write a small in-house `pgvector` skill.
4. `ai-evals-course/evals-skills` @ `80d5f7b0127c7572ed9e9339937adbfd7240ffeb`: eval-audit, evaluate-rag, error-discovery, write-judge-prompt, validate-evaluator, generate-synthetic-data.
5. `OWASP/secure-agent-playbook` @ `1b5fd4cff76075feb56d61ee2985e82516f8c53b`: prompt-injection-test, llm-risk-assess, code-review-security (CC-BY-4.0, attribution required).
6. `getsentry/skills` @ `d18b7aa8ba878354e5c348310230e652f7690f9c`: code-review, pr-writer, commit, gha-security-review, skill-scanner. Rewrite the Sentry-specific conventions, such as commit types, to match ours.
7. `docker/skills` @ `3e1cbd179989c2c193f3e4e6553a655907c2003b`: docker-project-foundations, docker-build-strategies, docker-compose-patterns, docker-destructive-guardrails.
8. `anthropics/skills` @ `8a1541c4a3ffa5a20a5a91de0dcf3f0bab1d1ef4`: skill-creator (for authoring in-house skills like pgvector, ADR, and ceitba-rag conventions).

Spec Kit: enable the official `git`, `bug`, and `assess` extensions. Pilot `adrkit` and `threatmodel` on a single feature before committing to them.

Gaps to fill in-house (small, owned skills): pgvector/hybrid retrieval tuning, the ADR template (unless adrkit wins), and the ceitba-rag docs style guide.

Pin strategy (vendor, don't install):

- Copy only the needed skill folders to `.kiro/skills/<upstream-skill-name>/` at a fixed commit SHA. Use no `npx skills add`, no plugin marketplaces, and no auto-update.
- Add `UPSTREAM.txt` to each vendored folder:
  ```
  repo: https://github.com/<org>/<repo>
  path: <path/in/repo>
  commit: <full SHA>
  license: <SPDX>  (upstream LICENSE copied alongside)
  vendored: <YYYY-MM-DD> by <handle>
  local-changes: none | <summary>
  ```
- Copy the upstream LICENSE/NOTICE into each folder. Keep CC-BY-SA material unmodified or clearly marked as modified.
- Before merging, review every `SKILL.md` and `scripts/` file by hand for network calls, shell execution, or instructions that override agent policy. Run Sentry's `skill-scanner` as a second opinion. Add `.kiro/skills/` to CODEOWNERS.
- To update, re-vendor at a new SHA in a dedicated PR that includes the upstream diff. Never track `main`.
- Spec Kit community extensions get the same treatment: pin to a release tag or SHA and review the source first.

## Sources

- [anthropics/skills](https://github.com/anthropics/skills) · [Agent Skills spec](https://github.com/agentskills/agentskills)
- [skills.sh](https://www.skills.sh/) · [Vercel: State of agent skills](https://vercel.com/blog/state-of-agent-skills) · [Ry Walker on skills.sh](https://rywalker.com/research/skills-sh)
- [trailofbits/skills](https://github.com/trailofbits/skills) · [trailofbits/skills-curated](https://github.com/trailofbits/skills-curated)
- [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang) · [cxuu/golang-skills](https://github.com/cxuu/golang-skills)
- [supabase/agent-skills](https://github.com/supabase/agent-skills)
- [ai-evals-course/evals-skills](https://github.com/ai-evals-course/evals-skills) · [Hamel: Evals skills](https://hamel.dev/blog/posts/evals-skills/)
- [OWASP/secure-agent-playbook](https://github.com/OWASP/secure-agent-playbook)
- [getsentry/skills](https://github.com/getsentry/skills) · [docker/skills](https://github.com/docker/skills) · [Docker Skills docs](https://docs.docker.com/ai/skills/)
- [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) · [obra/superpowers](https://github.com/obra/superpowers)
- [VoltAgent/awesome-agent-skills](https://github.com/voltagent/awesome-agent-skills) · [royalpinto007/awesome-agent-skills](https://github.com/royalpinto007/awesome-agent-skills)
- [Spec Kit community extensions](https://github.github.com/spec-kit/community/extensions.html) · [catalog.community.json](https://github.com/github/spec-kit/blob/main/extensions/catalog.community.json) · [community presets](https://github.com/github/spec-kit/blob/main/docs/community/presets.md)
- [mbeacom/adrkit](https://github.com/mbeacom/adrkit) · [NaviaSamal/spec-kit-threatmodel](https://github.com/NaviaSamal/spec-kit-threatmodel) · [ashbrener/spec-kit-red-team](https://github.com/ashbrener/spec-kit-red-team)

Content was rephrased for compliance with licensing restrictions.


## Vendored (2026-10-02)

I vendored all 8 adopted sources into `.kiro/skills/<name>/` from the SHAs recorded above. Each was fetched with a shallow fetch of that exact commit, and every SHA resolved. I used no npx, no marketplaces, and no Spec Kit extensions. Each folder has the upstream LICENSE (and NOTICE where one exists) plus `UPSTREAM.txt`, with `local-changes: none`. Every listed subfolder existed at its SHA. Plugin-wrapper files that sit inside the skill folders (`agents/openai.yaml`, `skill.yaml`, `evals/`) were kept verbatim. Hooks, commands, and agents outside the skill folders were not copied.

| Folder(s) | Source @ SHA | License | Review |
|---|---|---|---|
| `golang-error-handling`, `golang-context`, `golang-database`, `golang-testing`, `golang-security`, `golang-observability`, `golang-concurrency`, `golang-lint` | samber/cc-skills-golang @ `8e899e20ff0cd4dc524af3993e4c62d8ee8c5717` | MIT | Pass. Docs only, no scripts. `golang-security/references/secrets.md` has deliberate fake secrets (`AKIAIOSFODNN7EXAMPLE`) as "don't" examples. |
| `modern-python`, `property-based-testing`, `differential-review`, `sharp-edges`, `supply-chain-risk-auditor` | trailofbits/skills @ `82fe8226252622fa807643bdca1710901198553a` | CC-BY-SA-4.0 | Pass with notes. `modern-python` docs suggest `curl … \| sh` installers for uv/prek; prefer brew/pipx. `supply-chain-risk-auditor/scripts` queries public registries (OSV, npm, PyPI, proxy.golang.org, deps.dev, scorecard, GitHub API), which sends dependency names to those services. It also reads `gh auth token` and sends it only to `api.github.com`. Its bundled `test_*.py` are benign, and pytest skips `.kiro/` by default (dot-dir). The `sharp-edges` zero-width chars are ZWJ inside a Swift emoji example. |
| `supabase-postgres-best-practices` | supabase/agent-skills @ `544bfc56c89afe2b87b20017a59b2c6e9502a1fb` | MIT | Pass. Docs only. |
| `eval-audit`, `evaluate-rag`, `error-discovery`, `write-judge-prompt`, `validate-evaluator`, `generate-synthetic-data` | ai-evals-course/evals-skills @ `80d5f7b0127c7572ed9e9339937adbfd7240ffeb` | Apache-2.0 | Pass. Docs only. `write-code-eval` appears in the candidates table but not in the adopt list, so it was not vendored. |
| `prompt-injection-test`, `llm-risk-assess`, `code-review-security` | OWASP/secure-agent-playbook @ `1b5fd4cff76075feb56d61ee2985e82516f8c53b` | CC-BY-4.0 (+ THIRD_PARTY_NOTICES: Arcanum taxonomy CC-BY-4.0) | Pass. Docs only. |
| `code-review`, `pr-writer`, `commit`, `gha-security-review`, `skill-scanner` | getsentry/skills @ `d18b7aa8ba878354e5c348310230e652f7690f9c` | Apache-2.0 | Pass. `gha-security-review` and `skill-scanner` contain attack payloads and injection strings as reference material. `scan_skill.py` is a local regex scanner with no network access. `commit`/`pr-writer` still carry Sentry conventions; adapting them is TODO and will make `local-changes` non-empty. |
| `docker-project-foundations`, `docker-build-strategies`, `docker-compose-patterns`, `docker-destructive-guardrails` | docker/skills @ `3e1cbd179989c2c193f3e4e6553a655907c2003b` | Apache-2.0 | Pass. The `verify-*.sh` scripts only run `docker build`/`images`/`inspect` and `docker compose config`. |
| `skill-creator` | anthropics/skills @ `8a1541c4a3ffa5a20a5a91de0dcf3f0bab1d1ef4` | Apache-2.0 (+ repo THIRD_PARTY_NOTICES) | Pass with notes. The eval scripts shell out to the `claude -p` CLI, so they are Claude Code-specific and won't run under Kiro as-is. They also write temporary files into `.claude/commands/` (removed in a `finally` block). `eval-viewer/generate_review.py` serves on 127.0.0.1 and SIGTERMs whatever already listens on its port (default 3117). `viewer.html` loads Google Fonts and an SRI-pinned SheetJS CDN script. |

Review method: a manual grep of every `SKILL.md`, reference, and script for pipe-to-shell, credential paths, env/secret reads, network calls, subprocess use, eval/exec, HTML comments, and hidden Unicode. I also read every bundled script's network and subprocess paths line by line, then ran Sentry's `skill-scanner/scripts/scan_skill.py` on each folder as a second opinion. All scanner criticals were false positives: threat-pattern documentation, a `!` before `:` in commit-format prose, and "bypass security" inside a checklist. No skill was removed.

Other changes: added `.github/CODEOWNERS` (`/.kiro/skills/ @ceitba/maintainers`, a placeholder team). Added a `.gitignore` negation so `golang-testing/references/coverage.md` isn't swallowed by the `coverage.*` rule.
