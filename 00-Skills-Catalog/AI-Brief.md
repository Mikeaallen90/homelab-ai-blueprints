# Useful AI Skills Catalog — AI Brief

Read this, then ask the questions below before building anything. Goal: build the specific skills the user wants, adapted to their own setup — this doc gives you the generic logic; their actual paths, agent names, and notification channel will differ from any example here.

This catalog assumes the user already has at least one working agent ([Blueprint #3](../03-Agent-Creation/AI-Brief.md) — Agent Creation). Don't build skills #6–8 (which assume multiple agents) if they only have one; ask first.

## Questions to Ask Before Starting

1. Which skills from the list below do they actually want? (Show the numbered list from `Guide.md` — don't assume "all of them.")
2. Where do they want conversation transcripts/summaries and logs saved? (A vault, a plain folder, cloud storage — get a real path.)
3. If building Status Check / Update Check / Agent Conversation: how many agents do they have, what are they called, and how should the director/coordinator reach each one (direct dispatch, a subagent-style call, SSH, something else)?
4. Do they want alerts (Status Check down-alerts, Update Check major-version alerts)? If yes, what channel — Telegram, email, something else — and confirm the credential will be stored in a password manager ([Blueprint #2](../02-Vaultwarden-Password-Manager/AI-Brief.md)), not hardcoded in the skill file.
5. For Save Conversation / Close-Out: what naming convention do they want for saved files, and do they run more than one distinct agent/project that would need separate save locations tracked independently?

## Skill-by-Skill Build Logic

### 1. Save Conversation
- Resolve a save location (ask if not already known) for transcripts and summaries, plus a small state file recording the last save (name, paths, timestamp).
- If a previous save exists: ask whether to append new content since that save, start a new save, or update a different named save.
- New save: ask for a short name. Write a verbatim transcript (tool calls as one-line summaries only, never full output) and a condensed summary (topics, questions asked, outcomes, files changed, next steps).
- Append: find the last captured exchange in the existing transcript, append everything after it under a "Session Continued" divider, and regenerate the summary to reflect the whole conversation, not just the delta.
- If any part of the conversation was auto-compressed by the tool before this save, mark the reconstructed portion clearly rather than presenting it as verbatim.
- Always finish by invoking skill #3 (Logs) with a short description of what happened, then update the state file and confirm the paths to the user.

### 2. Close-Out ("the-end")
- Same save logic as #1, but never prompts the user for a title — pick one yourself from the conversation content.
- Support a "no save" argument that skips straight to closing without archiving anything.
- After saving (or after "no save"), find the actual terminal/session process this skill is running in and close it — but first check it isn't a shared terminal instance hosting other unrelated windows/tabs. If it is, refuse and explain why, rather than closing everyone's terminal.
- If not running in a closeable terminal context at all (e.g. plain SSH), just report there's nothing to close.
- The close action is always the last thing this skill does — print any confirmation before it, since nothing is visible after.

### 3. Logs
- Determine which log is "this session's own" based on context (which agent/project is currently active).
- Identify what actually changed this session and whether it's confirmed working or still pending — say so explicitly either way, don't overstate a fix as done if it isn't verified yet.
- **Before writing anything**, search each target log for an existing entry on the same topic. Nothing found → write a new entry. A prior entry exists and this adds new information → append a short dated addendum to it (or reference it explicitly) rather than creating a disconnected duplicate. A prior entry exists and there's nothing new to add → skip that target and say so.
- Write to: this session's own log, a different agent's log if the change actually belongs to their domain instead, and a shared cross-agent log if the user runs multiple agents and this is something others would need to know.
- If a shared cross-agent log exists, rotate it as part of this skill: rows from prior months move verbatim into a per-month archive file (keep the most recent ~10 rows live regardless of month; add an "older entries: see archive" pointer to the live table's intro once). This is the one sanctioned exception to append-only — rows are moved unchanged, never edited or dropped.
- Append-only — never rewrite or delete existing log history.

### 4. Load Logs
- At session start, read the last ~10 lines of this agent's own relevant log file(s) — don't read the whole file, that defeats the point of keeping this cheap and fast.
- If this is a coordinator/lead session across multiple agents, ask (once) whether to also pull the last few lines from other agents' logs.
- Report back briefly — which files were loaded, and flag anything that looks unresolved or pending a decision. This is context-loading, not a full status report.
- Read-only. Never edit or trim what you read here.

### 5. Scan Directory
- Glob the target directory recursively; identify file types, key files (README, config files like package.json/pyproject.toml), and directory structure.
- Produce a structured summary: file/folder counts, file-type breakdown, main directories and their apparent purpose, a guess at project type.
- For very large directories, summarize rather than listing exhaustively. Should run quickly — this is meant to be a fast orientation step, not a deep audit.

### 6. Status Check (Scheduled Self-Reporting)
- Requires: a director/coordinator agent, and at least one service-specific agent per thing being monitored.
- Fan out to each service agent (in parallel where the tooling allows it) with a consistent instruction: check your service, attempt a restart if it's down, report back {service, status: UP/DOWN/RECOVERED, notes}.
- Collect all results. If anything is still DOWN after a recovery attempt, send exactly one rolled-up alert — not one per service — via whatever channel the user chose. No alert at all if everything's fine.
- Log a one-line summary (date/time, N up, any down/recovered) to the director's own log.
- For the deeper architecture (including an independent second-machine watchdog for when the whole primary machine is down), point the user at [Blueprint #5](../05-Scheduled-Self-Reporting/AI-Brief.md) — this entry only covers the self-reporting half.

### 7. Update Check
- Same fan-out shape as #6, different question: check for an available update instead of health.
- Patch/minor version updates: apply automatically, log the result.
- Major version bump: do not auto-apply — write a fuller report for the user to review, and send an alert flagging it specifically as a major update needing a decision.
- Run on a slower cadence than Status Check (weekly is a reasonable default) since update-checking is less urgent than health-checking.

### 8. Agent Conversation (Start / Reply / End)
- Requires at least two agents that can each be invoked headlessly (a one-off call that returns a response, not necessarily an interactive session).
- **Start**: initialize a markdown file with a state block at the top — status, whose turn it is, a turn counter, a max-turn failsafe (default 8 if not specified), who owns writing the conclusion, and the exact command to hand off to the other agent. Write the first message, update the state block (turn +1, flip whose turn it is), then invoke the handoff command.
- **Reply**: read the file and state block, append a reply under a clear per-turn heading, update the state block the same way, hand off again — unless the turn limit or objective is reached.
- **End**: set a "ready to end" or "max turns reached" status and hand off to whichever agent is responsible for writing the actual conclusion — don't have the ending agent write the conclusion itself if a different agent owns that job.
- Never let both agents write to the file in the same turn (respect whose turn it is), and never silently loop past the max-turn failsafe.

## Watch For

- Never hardcode a credential (bot token, API key, password) into any of these skill files — pull it from a password manager at runtime ([Blueprint #2](../02-Vaultwarden-Password-Manager/AI-Brief.md)) instead. This applies most directly to #6/#7's alerting step.
- Keep #1 and #2 as separate skills — don't merge save-and-close into one, since that risks closing more sessions/windows than intended if the user runs several at once.
- #4 doesn't do anything on its own — it has to actually be called (or wired into a session-start routine) to have any effect. Confirm it's actually invoked somewhere, not just built and forgotten.
- #6/#7/#8 all assume a multi-agent setup already exists. Don't build them for a single-agent user — confirm agent count first.

## Done When

- Each requested skill exists as its own file, invocable by name.
- Any skill that saves/logs actually writes to a real, confirmed path (not a placeholder) — test it once, don't just assume the path is right.
- Any skill that sends alerts has been confirmed to use a password-manager-sourced credential, not a hardcoded one.
- If Status Check / Update Check / Agent Conversation were built, at least one real end-to-end run was completed (not just written and assumed to work) — a fan-out that actually reaches each agent, or a conversation thread that actually hands off correctly.
