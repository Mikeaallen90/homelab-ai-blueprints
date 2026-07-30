# Blueprint #8 — Email Archiving

## 0. Why This Exists

This project isn't for everyone, and that's fine. But if you're a hoarder of digital data, your Google (or other provider's) storage is full, you don't want to lose important emails, you run a business or otherwise need to keep old emails around, or your provider is warning you that you won't be able to send/receive soon — this shows you how to actually download and store them yourself. I use a NAS.

I downloaded my entire history in one shot, but I'd recommend going year by year instead — it makes the archive much easier to search later. Once it's downloaded, you can even build an agent to track and search through it for you. And beyond just archiving by date, having your AI help prune emails from senders or companies you don't actually need is worth doing too — anything that helps lower the size is welcome.

Once you've got a system in place, you can have a script run it automatically every year to keep your live inbox storage low.

## 1. What It Does

Downloads your full mailbox (or everything older than a cutoff) via IMAP to local storage, mirrors that to your own NAS/backup location, then deletes the archived messages from the provider — freeing up quota without losing anything. Repeatable on a schedule (yearly is reasonable for a personal inbox). Once archived, the mailbox becomes a plain folder of files you fully control — searchable directly, or by pointing an agent at it (see Adaptation Notes).

## 2. What You'll Need

- An email account with IMAP access enabled (Gmail, most providers support this)
- An **app password**, not your real account password — required once 2FA is on, and lets you scope/revoke script access separately from your login. Should live in a password manager (see [Blueprint #2](../02-Vaultwarden-Password-Manager/Guide.md), Vaultwarden), not a plaintext config file.
- `isync`/`mbsync` (free, open source) — or any IMAP sync tool
- Somewhere to put the archive: NAS, external drive, second cloud account, etc.
- A way to actually delete the old messages after backup is confirmed — IMAP delete via script (Python `imaplib`, or similar in any language)

## 3. How It Works

```
Provider (IMAP) --mbsync--> Local staging folder --rsync/copy--> NAS/backup storage
                                                                        |
                                                          (confirm backup intact)
                                                                        |
                                                              Delete originals via IMAP
```

Two-stage transfer (provider → local staging → final storage) rather than direct-to-NAS, so a network hiccup mid-sync doesn't touch the provider side, and the same staging folder can be checked over before anything gets deleted.

## 4. Step-by-Step Setup

1. Enable IMAP in your provider's mail settings.
2. Generate an app password and store it in a password manager, not a config file — you'll point mbsync at it via `PassCmd` in the next step.
3. Install `mbsync`; write a config (`~/.mbsyncrc`) pointing at your IMAP account and a local Maildir folder. Exclude the "all mail"/"everything" virtual folders providers like Gmail expose, since those duplicate every labeled folder. For the password: if your sync tool supports a `PassCmd`-style option (mbsync does), point it at a small helper script that fetches the password from your password manager's CLI at runtime instead of a plaintext `Pass` line — proven working end-to-end on mine. **Linux only:** if your sync tool runs under AppArmor confinement, its shipped profile likely won't allow executing a shell to run that helper command — you'll need to add an exception to the local override profile (see Gotchas).
4. **Linux only:** if AppArmor is enforced, add a local profile override allowing the sync tool read/write access to its staging folder — otherwise the sync silently fails or gets denied.
5. Pick your range and download it **one year at a time**, oldest first, rather than everything in one pass. It's slower up front, but it keeps each archive chunk small enough to actually search later instead of one giant undifferentiated dump. (I did mine as one big backup — year-by-year is the better call, especially with the download cap below.)
6. Expect provider-side rate limits (Gmail: ~2.5GB IMAP download per day) — even a single year can take more than one run for a heavy inbox; don't assume one sitting finishes it.
7. Copy the staged Maildir to your NAS/backup location, named by year (e.g. `Gmail-2024/`). Verify file counts/sizes match before touching the source.
8. Once verified, delete that year's range from the provider via IMAP (script or manual). Keep only "current" mail (e.g. this year) in the live account.
9. **Optional, separate from date-based archiving:** have an AI assistant go through your still-live inbox and flag (or delete, with confirmation) low-value senders — newsletters, one-off receipts, companies you no longer deal with. This shrinks your live mailbox on its own, independent of the yearly archive-and-delete cycle.
10. Repeat the year-by-year archive on a schedule — yearly, right after the new year starts, is reasonable for a personal inbox.

## 5. Adaptation Notes

- Provider swap: any IMAP-capable provider works the same way; only the IMAP hostname/port changes.
- Retention window: "keep current year live, archive everything older" is one policy — could instead be "archive anything older than N months" or triggered by storage-quota percentage instead of a calendar date.
- Automation: this can be scripted end-to-end (sync → verify → delete → notify) and scheduled via cron or a workflow tool (n8n) — see [Blueprint #3](../03-Agent-Creation/Guide.md)'s CLI-wrapper pattern if an agent should run/report on it.
- Once archived, don't let it just sit there — an agent with read access to the archive folder can search and summarize it on request, the same pattern used for the SMS/text archive ([Blueprint #9](../09-SMS-Archive-And-Search/Guide.md)). Year-by-year folders make this much more useful, since the agent (or you) can scope a search to a specific year instead of grepping everything.
- The AI-pruning step (removing low-value senders from the *live* inbox) is a genuinely different technique from date-based archiving — one shrinks by time, the other shrinks by sender/relevance. They can run independently or together.

## 6. Gotchas

- **App password in plaintext**: the straightforward mbsync setup puts your app password directly in a config file readable by anything running as your user. Route it through a password manager and inject it at runtime instead — mbsync supports this directly via `PassCmd "your-cli-helper here"` in place of the plaintext `Pass` line. If your distro confines the sync tool with AppArmor, the shipped profile likely denies the shell-exec `PassCmd` needs (mine did — it silently failed with "Permission denied" until I added an exception to `/etc/apparmor.d/local/<tool>` allowing the shell it invokes to run). Check your system's confinement logs (`journalctl -k`, look for `apparmor="DENIED"`) if `PassCmd` fails with no obvious cause.
- **Provider daily download caps**: a large mailbox can take multiple days to fully back up, even split by year; plan for that, don't assume one run finishes it.
- **Verify before you delete**: this is a one-way, provider-side-destructive step. Don't delete anything until the backup copy is confirmed complete and sitting somewhere durable (not just the local staging folder).
- **"Automate this" is easy to say and not actually do**: it's tempting to plan a yearly automated run and never build it — the schedule alone doesn't make it happen. Confirm the actual script/workflow exists and has run at least once, not just that it's been discussed.
- **AI-assisted deletion needs a real confirmation step**: if you let an AI flag low-value senders for removal, don't let it delete unattended the first few times — review what it picked before it becomes routine.
- **Status on the reference system**: the yearly automation itself hasn't been tested end-to-end yet — the first backup was done manually as one big pass rather than year-by-year, and a full year hasn't passed since to trigger a real automated cycle. This will get updated once an actual scheduled run has happened.
