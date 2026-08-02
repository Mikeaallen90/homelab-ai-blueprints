# Changelog

Notable changes to these blueprints, newest first.

## 2026-08-02 — `2eb840c`

### Changed
- **Blueprint 3 (Agent Creation)**: added guidance on picking a uniform agent-naming convention up front when multiple agents share one external trigger/caller (a phone automation tool, webhook, etc.), a gotcha on safe vs. unsafe agent-handle renames, and a gotcha that a "Creator" agent (one that builds other agents) should never edit blueprint content directly — it writes a handoff report instead, same as any other agent.

## 2026-08-01 — `0ce5bc4`

### Added
- **Blueprint 3 (Agent Creation)**: new "Repo Update Checker" bonus — a scheduled job that checks this repo for new commits and notifies you, so keeping up doesn't require manually checking back. Cross-referenced from Blueprint 5.

### Changed
- **Blueprint 2 (Vaultwarden)** and **Blueprint 3 (Agent Creation)**: added real-world gotchas from operating a password vault at scale (bulk-import duplication, folder structure at scale, hash-based verification, the vault's own master-secret backup problem) and from AI agents specifically handling secrets (safe vs. unsafe redaction, credential-routing rules for agent skills).
- **Blueprint 3 (Agent Creation)**: added gotchas to the phone remote-control bonus from a real webhook incident — a backgrounded launch command can report false success, external trigger fields need validation, and live workflow edits have a stacking limit before a restart is needed.

## 2026-07-30 — `c3fa81f`

### Fixed
- **Services Overview**: all 38 relative links were broken on GitHub (wrong relative-path depth for a root-level file).
- **Blueprint 3 (Agent Creation)**: 2 stale vault-only path references corrected.

## 2026-07-30 — `182749c`

### Added
- Initial publish — all 9 blueprints, Services Overview, and Skills Catalog. CC BY 4.0, public.
