# Useful AI Skills Catalog

## What This Is & Why It's Useful

If you've already built an agent (see [Blueprint #3](../03-Agent-Creation/Guide.md) — Agent Creation) and just want to add proven, reusable capabilities to it, this is the doc for that — you don't need to read the whole Agent Creation blueprint to use it. Each entry below is a generic, portable skill: a reusable procedure your AI CLI tool can be told to run by name, instead of you re-explaining the same steps in every conversation.

These aren't tied to any one service (unlike, say, a Spotify-download skill, which only makes sense once you've set up that specific pipeline — those live inside their own blueprints instead). Everything here works for basically any Claude Code-style agent setup, regardless of what else you've built. Some of this content also appears inside [Blueprints #3](../03-Agent-Creation/Guide.md) and [#5](../05-Scheduled-Self-Reporting/Guide.md), where it's discussed as part of building an agent from scratch — it's repeated here on purpose, so this file works as a standalone menu.

A quick note on why you'd want more than one of these: a running conversation isn't durable on its own — once a session ends, whatever wasn't written to a file is gone unless you deliberately resume it. **Saved conversations** preserve one specific session in full, for picking a task back up later. **Logs** are a quick, skimmable record of what's been going on, without rereading a whole transcript. Together they cover two different needs that a single `memory.md` file (durable facts only) doesn't.

---

## 1. Save Conversation

**What it does:** Archives the current conversation to two files — a full/verbatim transcript and a shorter summary — in a fixed location you choose, and offers to append to a previous save instead of always starting a new one.

**Why it helps:** Session history normally disappears once a conversation ends or its context window gets summarized. This makes it durable and searchable later — by you, or by another agent that needs to know what happened.

**How it works:** Checks whether a "last save" record exists; if so, asks whether to append new content to it, update a different named save, or start fresh. Writes the transcript verbatim (tool calls as one-line summaries, not full output), writes a condensed summary (topics, questions asked, outcomes, files changed, next steps), then updates logs (see #3 below) before confirming.

## 2. Close-Out ("the-end")

**What it does:** A deliberate session-ending skill — calls Save Conversation, then closes the terminal window/session it's running in. Accepts a flag to skip saving if you just want to close without archiving.

**Why it helps:** Gives you (or a remote-triggered agent) a clean, one-command way to actually end a session rather than just walking away from an open terminal. Kept as a separate skill from Save Conversation rather than merged, so a save-only mid-session checkpoint never accidentally closes a window you meant to keep open.

**How it works:** Picks its own title (no prompting), writes the transcript/summary the same way Save Conversation does, updates logs, prints a confirmation, then — as its last act — finds and closes the specific terminal process this session is running in. Includes a safety check so it never closes a shared terminal instance that's hosting other unrelated windows.

## 3. Logs

**What it does:** After any change worth remembering, writes an entry to every log that should know about it — this agent's own log, a different agent's log if the change actually belongs to their domain, and a shared cross-agent log if you run more than one agent and others would need to know.

**Why it helps:** Without this, knowledge of what changed lives only in one conversation's memory. This is what lets you (or another agent) ask "what's the status of X" and get a real answer later, instead of everyone having to re-derive it.

**How it works:** Figures out what changed and how confident/confirmed it is. Before writing anything, checks each target log for an existing entry on the same topic — if one exists, it appends a short dated addendum instead of creating a disconnected duplicate entry (this dedup check is the one piece of this skill not already described elsewhere — it's what keeps repeat calls from cluttering a log with restated information). Writes append-only, in whatever format that specific log already uses. If a shared cross-agent log exists, the skill also rotates it as a built-in step: at each month boundary, prior-month rows move verbatim into a per-month archive file, keeping the most recent handful live — a pruning rule that runs on every write instead of relying on someone remembering it (see [Blueprint #4](../04-Multi-Agent-Interaction/Guide.md)'s Gotchas for why that matters).

## 4. Load Logs

**What it does:** The read-side counterpart to Logs — at the start of a session, tails the last handful of lines from your relevant logs so the agent isn't starting blind.

**Why it helps:** Writing logs is only half the point — nothing reads them back automatically otherwise. This closes that gap deliberately, rather than relying on you to remember to ask "what happened last time" every session.

**How it works:** Reads the last ~10 lines of this agent's own log file(s). If you're running a lead/coordinator session across multiple agents, optionally offers to also pull the last few lines from other agents' logs, in case something cross-cutting is relevant. Read-only — never edits or trims what it reads.

## 5. Scan Directory

**What it does:** Quickly explores an unfamiliar folder or codebase and reports back a structured summary — file/folder counts, file type breakdown, key files (README, config files), and a guess at what kind of project it is.

**Why it helps:** A fast, general-purpose way to get oriented in a new directory before doing real work in it — useful for a new agent's first look at a project, or any time you (or an agent) needs context about a folder it hasn't seen before.

**How it works:** Globs the directory recursively, categorizes what it finds, and produces a short report rather than an exhaustive file-by-file listing (especially for large directories).

## 6. Status Check (Scheduled Self-Reporting)

**What it does:** A director/coordinator agent periodically directs each of your service-specific agents to check on their own service, attempt a restart if it's down, and report back. Only sends you one alert, rolled up, if something couldn't recover on its own — not one ping per service.

**Why it helps:** Once you have more than one self-hosted service, checking on all of them manually becomes a chore, and if you don't check, problems sit unnoticed. This is the fuller pattern — see [Blueprint #5](../05-Scheduled-Self-Reporting/Guide.md) for the complete architecture, including an independent-hardware watchdog backstop for when the whole machine running this is down. It's included here too so you can build just this piece without needing the rest of #5.

**How it works:** Fans out to each service agent in parallel (batches work well if you have several), collects UP/DOWN/RECOVERED results, sends a single alert only if something is still down after a recovery attempt, and logs a one-line summary.

## 7. Update Check

**What it does:** The same fan-out pattern as Status Check, on a slower cadence (weekly is typical), checking each service for available updates instead of health. Patch/minor versions apply automatically; a major version bump gets a full written report instead, since that's a decision worth a human looking at first.

**Why it helps:** Most self-hosted services don't update themselves. Without this, staying current is a manual per-service chore you have to remember to do.

**How it works:** Same director-fans-out shape as Status Check, but each agent's job is "check for an update, apply it if it's patch/minor, write a report if it's major" instead of a health check. Alerts only fire for major-version events.

## 8. Agent Conversation (Start / Reply / End)

**What it does:** A structured way for two of your agents to hold an asynchronous back-and-forth — one writes into a shared file, hands off, the other reads and replies, and so on — with a turn counter, a turn limit as a failsafe, and a clear rule for who's allowed to write the conclusion.

**Why it helps:** Useful once you have two or more agents that sometimes need to actually discuss something (debate an approach, hand off a task with context) rather than you manually relaying messages between them. The turn limit and single-writer-per-turn rules keep it from looping forever or both agents writing over each other.

**How it works:** A small state block at the top of a markdown file tracks whose turn it is, how many turns have happened, the max allowed, and who "owns" writing the conclusion. Starting the conversation initializes that block and writes the first message; each side's turn reads the file, appends its reply, updates the state block, and hands off (a headless call to the other agent) rather than looping in one session. Ending it sets a concluded status so neither side keeps replying past that point.

---

## Adaptation Notes

- All of these assume you already have at least one working agent ([Blueprint #3](../03-Agent-Creation/Guide.md)) — this catalog is about what to give it, not how to build it.
- File/save locations, agent names, and any messaging/alert channel (Telegram, email, etc.) shown here are placeholders — adapt them to wherever you actually keep notes and however you actually get notified.
- Skills #6–8 assume more than one agent exists. If you only have one agent, skip those for now — they become useful once you do.

## Gotchas

- **Never hardcode credentials (bot tokens, API keys) into a skill file.** Route them through a password manager (see [Blueprint #2](../02-Vaultwarden-Password-Manager/Guide.md)) instead — several of these skills involve sending alerts or making authenticated calls, and a skill file often ends up shared or version-controlled.
- Logs and Load Logs only help if something actually calls them — Load Logs isn't automatic just because it exists; it has to be invoked (or wired into your session-start routine) to do anything. For the write side, relying on the agent remembering to call Logs is the weak link — if your platform supports lifecycle hooks, the enforcement pattern in [Blueprint #4](../04-Multi-Agent-Interaction/Guide.md)'s Gotchas (a Stop hook that blocks session end when shared state changed but Logs never ran) is the mechanical fix.
- Keep Save Conversation and Close-Out as two separate skills, not merged — if you tend to run multiple sessions/windows at once, a combined save-and-close risks closing more than the one you meant to.
