# Agent Creation — AI Brief

Read this, then ask the user the questions below before doing anything. Goal: create a persistent, named AI agent — identity, memory, and (optionally) skills — that the user can return to across sessions, using a subscription-based CLI tool rather than a separately-billed framework.

## Context

Two patterns exist:
- **Directory-scoped**: a folder with `CLAUDE.md` (identity), `.claude/settings.json` (model pin), `memory.md` (long-term knowledge), optionally `Logs/` and `reports/`. The CLI tool auto-loads `CLAUDE.md` when its working directory is set to this folder.
- **Context-injected**: a single `.md` file whose content is prepended as the first message of a fresh session. No folder, no persistent memory file — for lightweight, stateless personas.

Skills are reusable procedures (`SKILL.md`: `name` + `description` frontmatter, then step-by-step body) invoked with `/skill-name`, either global or scoped to one agent.

A session's own conversation is not durable — it's gone once the session ends unless deliberately resumed. Explain this to the user: `memory.md` holds only distilled, durable facts; a saved conversation (full transcript + summary) is for resuming or finishing one specific interrupted task; `Logs/Status.md` is for quick situational awareness without rereading a transcript. All three matter, not just `memory.md`. See [00-Skills-Catalog](../00-Skills-Catalog/AI-Brief.md) for the skills that implement them.

This is distinct from — and should be described to the user as distinct from — any built-in "subagent" dispatch feature the CLI tool itself has for ephemeral single-task helpers. Named agents built this way are durable personas; internal subagents are usually short-lived task helpers. Don't conflate the two when writing the new agent's `CLAUDE.md`.

## Questions to ask before starting

1. What's this agent for, and does it need to remember things across unrelated conversations (→ directory-scoped) or is it a stateless persona for one-off dispatch (→ context-injected)?
2. What name/handle should it go by? If this agent will be one of several addressed by the same external trigger/caller (a webhook, a phone automation tool, a shared picker), check what naming convention the other agents already use and match it — don't pick a one-off slug in isolation. A single uniform rule (e.g. lowercase, no spaces/hyphens) across every agent lets the caller derive the right name with one transform instead of a growing lookup table.
3. What should its personality/tone be, and what are its responsibilities and boundaries (what should it explicitly NOT do)?
4. What existing files, folders, or systems should it know about / read for context?
5. Does it need any skills beyond the two default ones below? If so, what should each one do?
6. **Do you also want Telegram chat access and/or phone remote-control access set up for this agent?** If yes to either, use the Bonus sections in the accompanying `Guide.md` for the steps. If no, skip both and just confirm the base agent works.
7. If Telegram access is wanted: confirm the bot token will be stored in Vaultwarden ([Blueprint #2](../02-Vaultwarden-Password-Manager/AI-Brief.md)) and fetched via `bw`, not hardcoded anywhere — don't proceed with a hardcoded token even if the user doesn't raise it themselves.
8. **Do you want a Repo Update Checker set up** — a scheduled job that notifies you when the blueprints repo itself gets updated? If yes, confirm a notification channel exists (Telegram from Q6, or fall back to a log/note) and the default monthly cadence, or ask what cadence they'd prefer.

## Steps to execute

1. Create the folder (directory-scoped) or the single `.md` file (context-injected) per the answers above.
2. Write `CLAUDE.md` (or the equivalent identity file): identity, personality, responsibilities, boundaries, what to read for context, and an explicit note distinguishing this agent from the tool's internal subagent mechanism.
3. Directory-scoped only: add `.claude/settings.json` with an explicit `"model"` — never leave this unset and rely on a global default.
4. Directory-scoped only: create `memory.md` for long-term knowledge.
5. Build the two default skills (save-conversation-style, close-out-style — see Guide.md's "Useful Skills to Build First") unless the user says they don't want them, plus any custom ones requested. Keep save and close-out as two separate skills, not merged — protects against closing multiple open sessions/tabs at once. The save-conversation skill must include a step to update relevant logs (this agent's own `memory.md`/`Logs/` folder, plus any shared team log it participates in per [Blueprint #4](../04-Multi-Agent-Interaction/AI-Brief.md)) with what happened in the session — not just archive the transcript.
6. Test: launch the CLI tool with cwd set to the agent's folder (or inject the identity file for context-injected) and confirm it responds in-character and can find its own reference files.
7. **Directory-scoped only — build the CLI wrapper** (Guide.md Step-by-Step #8): a small executable named after the agent's handle, on `PATH`, that `cd`s into the folder and execs the CLI tool with passed-through args. Confirm the wrapper's folder is actually on `PATH` (check `.bashrc`/`.profile`), not just that the file exists. This is what makes "type the agent's name from anywhere" actually work — don't skip it even if the user didn't ask for it by name, since it's foundational to the pattern, not optional bonus material like Telegram/remote-control.
8. If Telegram and/or remote-control were requested, follow the corresponding Bonus section in `Guide.md` — both call into the wrapper from step 7 rather than duplicating the folder-launch logic.
9. If the Repo Update Checker was requested, follow its Bonus section in `Guide.md` — same scheduled-job mechanism as [Blueprint #5](../05-Scheduled-Self-Reporting/AI-Brief.md), pointed at the GitHub API instead of a local service.

## Watch for

- Don't let the agent inherit an expensive default model silently — the model pin must be explicit.
- Don't hardcode any credential (bot tokens included) into `CLAUDE.md`, `memory.md`, or any script this agent uses — route through Vaultwarden/`bw` ([Blueprint #2](../02-Vaultwarden-Password-Manager/AI-Brief.md)).
- If Telegram access is set up, confirm a sender-ID allowlist is in place before considering it done — an unrestricted bot is a real exposure, not a theoretical one. Check the numeric `from.id`, never the display name (spoofable), and enforce it in one central place rather than per-workflow if there's more than one bot.
- If a local HTTP proxy bridges the workflow tool to the agent, check what it's actually listening on and what network segments can reach it — proven on the live system that "just for the workflow tool to reach it" often means "reachable by anything on the LAN/other VLANs" unless explicitly firewalled down.
- If persistent conversation (across separate CLI invocations) is wanted, this requires deliberately capturing and reusing a session ID — it does not happen automatically just by reusing the same folder.
- If the Repo Update Checker is set up, don't poll faster than monthly without a real reason — GitHub's unauthenticated API is capped at 60 requests/hour per IP, and there's no upside to checking a docs repo more often than that.
- If this agent joins an existing group of agents reachable by name through the same trigger mechanism, confirm its wrapper/param name follows the same convention as the others' — an inconsistent one forces a lookup table on whatever's calling in, and that table silently goes stale as agents are added or renamed.
- If you rename an existing agent's handle, first confirm whether the thing that launches it derives the wrapper name dynamically (safe, one-line rename) or has it hardcoded somewhere (needs updating in that config too) — then sweep every doc/log/skill file that names the old handle so nothing goes stale.
- Check a command's documented output mode before running it against a secret or secret-adjacent value — does it return the raw value into something capturable, or does it print/dump it? Don't assume based on how a similarly-named flag behaved on a different tool; this applies to any new credential-adjacent tool the agent is given, not just `bw`.
- Never preview a file's content "for context" once it's already confirmed to hold a real secret — no redaction regex covers every format. Use line-number-only blind replacement and verify success via hash/boolean check, not by reading the result. Never redact by a fuzzy "this looks short/suspicious" heuristic either — only by an exact known value (string match) or a line already independently verified as a real secret; a wrong guess in either direction either leaks a value or destroys real, unrelated content.
- Route every credential lookup through the password-manager agent/vault — never let a script or another agent's file hardcode one, including bot tokens, webhook URLs, and API keys, not just logins. When a skill spots a credential that needs storing, don't have it guess whether the value is already in the vault or new — hand it to the password-manager agent with context and let that agent check first, to avoid duplicate/cluttered vault entries. If that agent is slow to respond and the calling skill can't pause to ask the user (e.g. a fully automated close-out flow), don't block indefinitely — write the safest available placeholder/pointer, flag the credential as pending confirmation, and move on.
- Build the "don't write secrets in plaintext" rule directly into whatever skill saves conversations, writes logs, or updates memory files — a one-time cleanup pass does not stay clean on its own. Cover one-line tool-call summaries too, not just message text (a careless "found API key: abc123" one-liner defeats the point).
- If a webhook or trigger backgrounds a launch command (`nohup ... &` or equivalent), never let that be the last thing before the webhook responds — gate it with a synchronous existence/success check first and branch the response on that check's own stdout, or "webhook returned success" and "the thing actually happened" can silently diverge with no log trail to catch it.
- Trim and validate any field from an external trigger source (phone automation tools especially) in the workflow itself — don't assume it arrives clean just because the source feels trusted. Format-only validation isn't the same as confirming the target actually exists.
- If a workflow serving a webhook has had multiple same-day live edits, restart the container before considering the setup done rather than assuming the hot-reload path stays clean indefinitely.

## Done when

- The agent responds in-character from a fresh session, using its `CLAUDE.md`/identity file.
- Directory-scoped: `memory.md` exists and the agent can read/reference it.
- The two default skills exist and work when invoked.
- Directory-scoped: the wrapper command works from a directory other than the agent's own folder, confirming `PATH` is actually set up correctly.
- If Telegram/remote-control were requested: a real end-to-end message or session was completed via that channel, not just the base agent tested locally.
- If the Repo Update Checker was requested: the scheduled job runs, correctly stores/compares the SHA, and a forced test (manually editing the stored SHA to something stale) triggers a real notification.
- Any skill that writes conversation transcripts, summaries, or logs has an explicit rule against writing real credential values in plaintext — checked, not assumed.
- If remote-control was set up via a webhook: a backgrounded launch step has a synchronous existence/success check gating the webhook's response, any external-trigger fields are trimmed/validated, and a restart has happened since the last of any multiple same-day live workflow edits.
- If this agent is one of a group reachable by the same external trigger, its param/wrapper name follows the same naming convention as the rest of the group — confirmed by inspection, not assumed.
