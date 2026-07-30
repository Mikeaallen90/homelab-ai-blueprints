# Blueprint #9 — SMS/Text Message Archive + Search Agent

## 0. Why This Exists

This project may not be needed by all. But if you've ever wondered whether you handled a conversation right, worried you misunderstood something, or needed to talk through a conversation without bringing another person into it — this is for you.

With the SMS backup app, you can set it up to back up your texts automatically. I suggest backing up every 4 hours or longer, using incremental backups, so you don't take up too much storage — but there are other options depending on how often you want fresh data. In that app, you can send your backups to Google Drive.

Now here's the key to the project: your storage unit. You need to verify the connection between where your SMS backups land and where your AI can actually reach them. I use a Synology NAS, and it has a Cloud Sync feature that pulls my Google Drive down automatically. Storing the backups somewhere your AI can access is what gives you the ability to create an agent to keep track of things and actually talk about them with you.

I also suggest having the agent's data live in a real database instead of raw files — it's faster and easier to search through. You can automate scripts to run off your backup uploads, so the whole pipeline just runs itself. That way, you can launch an agent and start talking about something or someone whenever you want.

This isn't for everyone, and I'm sure there are variations on how to build it.

## 1. What It Does

Continuously backs up your phone's SMS/MMS history to a personal SQLite database that lives on your own machine — never depends on the phone still having the messages, never depends on the carrier. Handles both a full-history rebuild and small ongoing incrementals automatically, dedupes on re-import (safe to reprocess anything), and gives you (or an AI agent) a queryable archive: pull any contact's full conversation, search across everyone, or get an honest read/summary of how a conversation actually went.

## 2. What You'll Need

- **SMS Backup & Restore** (free, Android, Play Store) — the actual export app: [play.google.com/store/apps/details?id=com.riteshsahu.SMSBackupRestore](https://play.google.com/store/apps/details?id=com.riteshsahu.SMSBackupRestore). Configure it for scheduled incremental backups (every 4 hours or longer is a reasonable default — long enough to keep storage down, short enough that the archive stays current) plus a periodic full backup, exporting to your cloud drive of choice (Google Drive, etc.)
- A way to get those files off your phone's cloud drive and onto storage your AI can actually reach — this is the piece that makes or breaks the whole project (see Step 2 below)
- Python 3 (stdlib only — no extra packages needed for parsing/importing)
- SQLite (bundled with Python)
- A machine that's on often enough to catch new exports on a schedule (cron, or systemd timer if on Linux)
- Optional: an AI agent with read (or read/write) access to the database, if you want conversational search on top of it, not just raw SQL

## 3. How It Works

```
Phone (SMS Backup & Restore) --cloud sync--> Cloud Drive --sync--> Your storage (AI-reachable)
                                                                        |
                                                        Watcher polls for new export files
                                                                        |
                                                    Wait for file size to stabilize (sync finished)
                                                                        |
                                              Split into safe chunks (large exports can choke a single pass)
                                                                        |
                                          Import into SQLite, dedup by content hash — safe to re-run
                                                                        |
                                    Classify full vs. incremental from the XML, clean up accordingly
                                                                        |
                                          Agent queries the database on request
```
Two backup types matter here: the export app typically runs **incremental** backups on a short interval (new messages only) plus a **full** backup on a longer interval. Incrementals get deleted from storage right after import (already safely in the DB); the most recent full backup is kept as a genuine from-scratch fallback, since the archive could in theory be rebuilt from it alone.

## 4. Step-by-Step Setup

1. Install SMS Backup & Restore on the phone; set it to back up SMS+MMS on a schedule (every 4 hours or longer, incremental, plus a periodic full backup), exporting to a cloud drive.
2. **This is the key step — verify the connection between your backup storage and your AI.** Whatever machine your AI agent actually runs on needs a way to reach the exported files automatically, not just a folder you'd have to check by hand. If your NAS or storage box has a cloud-sync feature, point it at the same cloud drive the phone app exports to, so files land somewhere your AI's machine can read without you moving anything manually. Confirm this actually works (a test file syncing all the way through) before building anything on top of it.
3. Write (or adapt) an importer: parse the XML export, extract sender/contact/date/body per message, skip binary media parts (photos/videos) but note their presence, and insert into a real database (SQLite is enough) — using a content hash (contact + date + sender + body) as a unique key so re-importing the same file twice is a no-op. A database beats raw files here — it's what makes searching fast and easy later.
4. Write a splitter for large exports: break the file into safe chunks at top-level message boundaries only (never mid-message), since a very large single import can get killed by resource/time limits depending on your environment.
5. Write a watcher script: poll the sync folder for new export files, confirm each has finished transferring (size stable across a wait interval, not just "file exists"), then split + import.
6. Add classification logic to that same watcher script (not a separate script): classify each processed file as full vs. incremental (the export format usually tags this) and clean up accordingly — delete incrementals once imported, keep only the latest full backup as a fallback.
7. Schedule the watcher (systemd timer or cron) to run every 10–15 minutes, with boot-persistence so it catches up if the machine was off. This is what makes the whole pipeline run itself off your backup uploads, with nothing manual after setup.
8. Point an AI agent at the database (read-only to start) so you can launch it and just start talking about a person or a conversation, instead of writing raw SQL every time.

## 5. Adaptation Notes

- iPhone: there's no direct equivalent to SMS Backup & Restore's local-export model — iMessage/SMS history typically needs to come from an iCloud/Messages-in-the-Cloud export or a third-party tool; the database/import/watcher shape here still applies once you have XML/JSON/CSV to parse, just swap the parser.
- No NAS with cloud-sync: any folder your cloud drive's desktop client syncs to locally works the same way — you don't need a NAS specifically, just somewhere the files land automatically that your AI's machine can also see.
- Backup interval: 4 hours is a starting point, not a rule — shorter means fresher data and more, smaller incremental files; longer means less storage churn but a bigger gap if you need something very recent.
- Retention: keeping only the latest full backup (not every one) is a size choice — keep more history if you want extra redundancy and have the space.
- If you don't want an AI layer at all, the SQLite database is directly queryable with any SQL client or a simple script — the archive is useful on its own.

## 6. Gotchas

- **Don't naively grep for the full/incremental tag** — if your export format allows metadata like a stylesheet tag near the top of the file, a loose text match can match the wrong thing. Anchor to the actual root element, not just "the first line containing this substring."
- **"File exists" isn't "file finished syncing"** — cloud sync can leave a partially-written file visible before it's complete. Check that file size is stable across a wait interval before processing, or you'll import a truncated/corrupt export.
- **Large exports can silently die mid-import** — worth chunking large files defensively even if a single-pass import seems to work on smaller ones; the failure mode encountered here wasn't an OS/memory issue, it was an execution-time ceiling in the automation environment.
- **The storage-to-AI connection is the one thing worth testing before anything else** — if that link doesn't actually work (wrong sync direction, wrong account, permissions), nothing built on top of it will either, and it's much easier to catch that with one test file than after building the whole importer around it.
- **Give an AI agent write access to the database deliberately, not by default** — start read-only. If write access is granted later (e.g. to let it fix a stale row), require a DB backup before any non-trivially-reversible edit, and keep it out of the pipeline's actual design decisions (schedule, retention policy) without checking with you first.
- **Privacy**: this is your full, permanent text history in one file. Treat the database file itself like a sensitive document — backup encryption, file permissions, and who/what has access to the machine all matter more once this exists.
