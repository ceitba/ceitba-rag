# 0012. Sync: per-connector change detection, soft deletes, force-sync cooldowns

- Status: Accepted
- Date: 2026-10-02
- Research: [04](../research/04-ingestion-sync-infra.md)

## Context
Sources: Google Drive (public or shared with our account), OneDrive (share links), GitHub public
repos. The source registry (users and their linked folders) is served by the CEITBA backend.

## Decision
- Uniform `Connector` interface: `list_changes(cursor) → [FileChange]`, `fetch(file) → bytes`.
- Change keys:
  - Drive: service account; `changes.list` for shared folders, recursive `files.list` with
    resource keys for public links; `sha256Checksum`/`md5Checksum`, `version` for Google-native files.
  - OneDrive: Entra app + team Microsoft account; `/shares/u!{b64}/driveItem`, delta; `cTag`.
  - GitHub: `git ls-remote` HEAD SHA, blobless clone, `git diff --name-status`.
- Two-level gating: upstream key decides download; our sha256 decides reprocessing.
- Upstream deletion → `stale` (still listed with a warning) → `deleted` after N weeks → GC.
- Weekly cron Sunday 03:00 America/Argentina/Buenos_Aires.
- Force sync via query-api: owners may sync only their own sources with a cooldown (default 7
  days, configurable); admins may sync any source or all. 202 queued / 409 in flight / 429 +
  `Retry-After`. Cooldown is enforced atomically in SQL.

## Consequences
First full Drive mirror takes ≥ 2 days (1 TB/day egress cap). OneDrive delta on other users'
personal drives needs a spike before committing.
