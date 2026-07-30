# Blueprint #8 — Email Archiving (AI Brief)

## Purpose

Build a repeatable process that downloads a user's mailbox via IMAP, verifies it's safely mirrored to their own storage, then deletes the archived range from the provider to free up quota — done year by year rather than all at once, so the resulting archive stays searchable. See `Guide.md` for the full human-readable version; this file is for an AI assistant executing the setup on the user's behalf.

## Questions to Ask Before Starting

1. What email provider, and is IMAP already enabled? (For Gmail: Settings → Forwarding and POP/IMAP.)
2. Does the user have or want to generate an app password? Confirm where it should be stored — strongly push for a password manager over a plaintext config file if one is available (see [Blueprint #2](../02-Vaultwarden-Password-Manager/AI-Brief.md), Vaultwarden).
3. Where's the final archive destination — NAS, external drive, second cloud account? Confirm it's actually reachable/mounted before starting a sync.
4. What year(s) should be archived first? Default to oldest-first, one year at a time — don't default to "everything at once" even if the user doesn't mention it, since that produces a much harder-to-search dump. If the user explicitly wants a single bulk backup anyway, that's their call, but flag the searchability tradeoff first.
5. Does the user want deletion from the provider automated after each year's backup is verified, or do they want to review/delete manually the first time?
6. Do they want the AI-assisted low-value-sender pruning step (Guide.md Step 9) as well, or just the date-based archive? These are separate techniques — confirm which one(s) they actually want before building both.
7. Do they want this scheduled to run automatically going forward (cron, or a workflow tool like n8n), or run on-demand only for now?
8. Do they eventually want a search/summarize agent pointed at the finished archive? Not required to build this blueprint, but worth flagging as a natural follow-up (parallel to [Blueprint #9](../09-SMS-Archive-And-Search/AI-Brief.md)'s SMS-archive-plus-agent pattern) so folders get organized in a way that supports it from the start (i.e., year-by-year, not one giant folder).

## What "Done" Looks Like

- IMAP sync successfully pulls a full year's mail into a local staging folder, matching what's actually in the provider account for that range.
- The staged files have been copied to the user's chosen storage, and file counts/sizes were verified to match before any deletion happened.
- The app password lives in a password manager (or the user explicitly declined and understands the plaintext tradeoff), not hardcoded in a committed or shared config file.
- The archived year's messages are confirmed deleted from the live provider account only after the backup copy was verified.
- If scheduling was requested: a script or workflow exists that can run the whole cycle (sync → verify → delete → notify) for the next year without manual babysitting, and it's actually been run at least once successfully — not just written and assumed to work.

## Key Points to Carry Into the Build

- Archive year by year, not all at once, even though a single bulk pass is technically simpler — this is a direct, explicit user preference (searchability later matters more than convenience now).
- Gmail-style providers cap IMAP downloads (~2.5GB/day) — a single year of a heavy inbox can still take more than one sitting. Don't assume a sync that stops early has failed; check if it's a rate limit first.
- Exclude a provider's "all mail"/virtual aggregate folders from the sync config — they duplicate every labeled folder and roughly double the download for no benefit.
- Deletion from the provider is one-way and destructive. Never trigger it until the destination copy has been positively verified (count/size check at minimum) — don't take "the sync command exited 0" as sufficient confirmation on its own.
- If AppArmor (or an equivalent Linux MAC system) is enforced, the sync tool may need an explicit local profile override to read/write its staging folder — a silent permission denial can look like a sync bug if this isn't checked first. This extends to `PassCmd`: if the password is fetched via a helper command instead of a plaintext config line, the profile also needs to allow the sync tool to exec a shell — proven necessary on the live system (mbsync's shipped AppArmor profile denied `dash` exec until a local override was added). Check `journalctl -k` for `apparmor="DENIED"` lines if a `PassCmd`-style fetch fails with no obvious cause.
- The AI-pruning step (deleting low-value senders from the *live* inbox) is a different mechanism from date-based archiving — don't conflate the two, and never let deletions from this step run unattended before the user has reviewed at least one round of what would be removed.
- If a search/summarize agent gets built on top of the finished archive later, year-by-year folder structure is what makes that practical — worth mentioning to the user even if they don't want the agent built right now.
