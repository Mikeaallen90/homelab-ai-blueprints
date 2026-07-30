# Blueprint #9 — SMS/Text Message Archive + Search Agent (AI Brief)

## Purpose

Build a pipeline that automatically backs up a user's SMS/MMS history from their phone into a searchable local database, then optionally puts an AI agent on top of it that can pull up or discuss any conversation on request. See `Guide.md` for the full human-readable version; this file is for an AI assistant executing the setup on the user's behalf.

## Questions to Ask Before Starting

1. Android or iPhone? SMS Backup & Restore (Play Store) is the direct path on Android — there's no equivalent one-app solution on iPhone, so confirm what export format is actually available before assuming this guide's importer applies as-is.
2. Where does the phone export to (Google Drive, another cloud drive), and what machine will the AI agent actually run on? **Confirm and test the sync path between those two before writing any importer code** — this is the single point of failure for the whole project. If there's no existing sync mechanism between the cloud drive and the AI's machine, that has to be set up first.
3. What backup interval does the user want? Default to every 4 hours, incremental, with a periodic full backup — but this is a size/freshness tradeoff, not a fixed rule, so ask if they want something different.
4. Does the user want an AI agent layered on top of the database, or just the archive itself (queryable by hand)? If an agent, what should its scope be — just this contact/conversation data, or does it also need other context about the user? Keep it narrow if unclear.
5. If an agent is wanted: read-only or read/write access to the database? Default to read-only. Only grant write if the user explicitly wants the agent able to fix/clean data itself, and if so, require a backup-before-edit rule and keep pipeline design (schedule, retention) off-limits to the agent without checking with the user first.
6. How much history does the user want kept — everything, or a rolling window? This affects retention policy for full backups, not just incrementals.

## What "Done" Looks Like

- The export app is configured and has produced at least one real backup file (full and, ideally, one incremental) that lands in a location the AI's machine can actually read — verified with a real test file, not assumed.
- A watcher process is running on a schedule (not manual), correctly detects new export files only after they've finished transferring (stable file size), and successfully imports them.
- The database has real message data in it, deduped correctly (re-running an import on the same file doesn't create duplicate rows), and full vs. incremental files are classified and cleaned up correctly (incrementals deleted after import, most recent full kept as fallback).
- If an agent was requested: it can query the database and hold a real conversation about a specific contact's history when asked, with the access level (read-only vs. read/write) matching what the user actually agreed to.

## Key Points to Carry Into the Build

- The storage-to-AI connection is the load-bearing piece of this whole project — test it in isolation (one file, manually placed, confirm the AI's machine can see it) before writing any parsing/import code. Everything downstream assumes this link works.
- Don't naively text-match for a "full vs. incremental" tag in the export XML — some export formats have unrelated metadata (like a stylesheet declaration) earlier in the file that can false-match a loose grep. Anchor to the actual root element.
- A file appearing in the sync folder doesn't mean it's finished syncing. Check that its size is stable across a wait interval (not just that it exists) before processing it, or a partial/corrupt file gets imported.
- Chunk large export files defensively before importing, even if a single-pass import works fine on smaller test files — a single very large import can hit an execution-time ceiling in some automation environments that has nothing to do with available memory.
- Use a content hash (contact + date + sender + body, or equivalent) as a database unique key so the importer is safe to re-run against the same file without creating duplicates.
- Default any agent built on top of the database to read-only. If the user wants write access later, that's a deliberate upgrade with guardrails (backup before edits, no unilateral pipeline-design changes), not the default.
- This is the user's full, permanent private text history in one file — treat the database itself as sensitive (file permissions, backup encryption, access control on the machine it lives on).
