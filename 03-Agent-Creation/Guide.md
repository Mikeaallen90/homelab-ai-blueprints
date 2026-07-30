# Agent Creation — Identity, Memory, Skills, Directory-Scoped Structure

## Why This Exists

Things like Hermes are great, but they can burn through tokens — especially the more skills your agent has. I developed a similar approach, but using what we already pay for (if you use a subscription). It may be slightly different from model to model, but Claude uses .md files, and each "Claude" has its own .md and memory. You'll want to make clear to your agent the difference between these created agents and its own internal subagents — but the AI can name them and set things up so you can type the name, like "Steve," and it'll pull up that agent. Having logs will keep things from being forgotten, and you can assign an agent specific tasks — one for each service you run to track and fix things, a personal agent, some for security, etc. I even have a writing team that helps me with my novel. If you've tried Hermes or OpenClaw, this setup gives you that same feel. You can also talk to these agents in multiple ways — I use the Telegram setup for quick one-time messages, like checking on or updating something. But the remote-control setup is the one I'm most proud of, because it lets you talk to the agents in the app, giving you that familiar chat feel. There's also a list of skills I believe will help.

## What It Does

Gives an AI CLI tool (Claude Code, or anything similarly structured) a persistent identity: who it is, what it knows, what it's responsible for, and where its memory lives — defined entirely in plain files, no code required. If you're already paying for a subscription (rather than metered API usage), this approach uses that existing plan instead of adding a separate token-metered framework on top, which is where tools like Hermes or OpenClaw can get expensive as an agent's skill count grows.

**Common patterns people build this way**: one agent per service/system you want monitored and self-fixing, a personal-assistant agent for day-to-day tasks, a dedicated security/watchdog agent, or a multi-agent "team" of specialists collaborating on one project (e.g. separate roles for a creative writing project).

## What You'll Need

- Claude Code (or an equivalent CLI-based AI agent tool that reads a local instructions file)
- A folder per agent, if using the directory-scoped pattern (below)

## How It Works

Two patterns, and most systems end up using both:

**Directory-scoped agents** — a dedicated folder contains a `CLAUDE.md` (the agent's identity: personality, responsibilities, what to read, how to behave) plus a `.claude/settings.json` (pins the model — see Gotchas) plus the agent's own working files: a `memory.md` for long-term knowledge, a `Logs/` folder for status/update history, a `reports/` folder for point-in-time deliverables. Launching the CLI tool with its working directory set to this folder makes it *become* that agent — it auto-loads `CLAUDE.md` as context. This is the pattern for agents with real ongoing state (a NAS agent that remembers past incidents, a personal assistant that remembers your schedule).

**Context-injected agents** — a single `.md` file (identity + instructions, no folder, no persistent memory file) that gets prepended as the first message to a fresh session. No dedicated folder needed. This is the pattern for lightweight personas — ones dispatched by an orchestrator, or that don't need their own long-term state, just a consistent voice and job description.

**Skills** — reusable procedures stored as `SKILL.md` files (YAML frontmatter with `name` + `description`, then a plain-language step-by-step body) in a skills folder. Invoked with `/skill-name`. These are shared capabilities any agent can be told to use — "save this conversation," "check service status" — without re-explaining the procedure inline every time.

**Memory** — separate from an agent's own `memory.md` (its personal knowledge file), there's a system-level memory pattern: a fixed folder with an index file (`MEMORY.md`) linking out to individual typed memory files (facts about the user, standing feedback/corrections, project state, external references). Built up incrementally over time rather than written once.

**Why you need saved conversations and logs too, not just memory.md** — A session's own conversation isn't durable: once it ends, or the context window fills and gets summarized, anything that wasn't written to a file is gone unless you deliberately resume that exact session. Three different layers cover three different needs: a **saved conversation** (full transcript + summary — see Useful Skills below) is for picking a specific interrupted task back up or finishing something later, with the full detail of exactly what was said; **`Logs/Status.md`** is a running, quickly-skimmable record so an agent (or you) can tell what's been going on lately without rereading a full transcript; **`memory.md`** holds only the distilled, durable facts that should shape behavior going forward, not a play-by-play. See `Blueprints/00-Skills-Catalog/` for the actual skills that implement all three (`save-conversation`, `logs`, `load-logs`).

**Calling it by name from anywhere** — a directory-scoped agent normally only "activates" when you launch the CLI tool with its folder as the working directory. A small wrapper script fixes that: a short executable on your `PATH` that `cd`s into the agent's folder and then execs the CLI tool, passing through any arguments. Typing the agent's name from *any* terminal, in any directory, launches (or headlessly messages) that agent — this is what makes "just type Steve" actually work, and it's the same mechanism a chat-bot or remote-control layer calls into under the hood.

## Step-by-Step Setup

1. Decide directory-scoped vs. context-injected for this agent (see Adaptation Notes for how to choose).
2. Give the agent a short name/handle (e.g. "Steve") — pick something you'll actually type or say. This becomes the wrapper command name (step 8) and the key any chat/remote-control layer uses to find the right folder or file.
3. **Directory-scoped**: create the folder, write `CLAUDE.md` inside it covering: who the agent is, its tone/personality, what it's responsible for, what files/paths it should read for context, and any boundaries (what it should *not* do). Add `.claude/settings.json` with an explicit `"model"` key.
4. **Context-injected**: write a single `.md` file with the same identity content — no folder needed.
5. Give the agent a `memory.md` (directory-scoped only) for anything it should remember long-term outside of a single conversation.
6. Build any skills it needs as `SKILL.md` files with a `name`, `description`, and step-by-step body; the description is what the tool uses to decide when a skill is relevant, so make it specific. See "Useful Skills to Build First" below for two worth having from day one.
7. Test by launching the CLI tool with working directory set to the agent's folder (directory-scoped) or by prepending the identity file to a fresh prompt (context-injected).
8. **Wrap it so you can call it by name from anywhere** (directory-scoped agents — this is the missing piece that makes the pattern actually feel like Hermes/OpenClaw):
   - Create a small executable script named after the agent's handle (e.g. `steve`), in a folder on your `PATH` (e.g. `~/.local/bin/`):
     ```bash
     #!/usr/bin/env bash
     cd "/path/to/agent/folder" || exit 1
     exec claude "$@"
     ```
   - `chmod +x` it. Now `steve` from any directory launches an interactive session as that agent, and `steve -p "check the logs"` sends it a one-off headless message from a script or another tool.
   - Optional: add a capitalized symlink (`ln -s steve Steve`) if you want the name to work either way you type it.
   - If the agent's folder lives on a network share or removable mount, have the wrapper check the folder exists first and attempt a remount if not — a wrapper that just silently fails when the mount drops is a worse experience than one that self-heals or gives a clear error.

## Useful Skills to Build First

Two small skills that pay for themselves almost immediately:

- **A "save conversation" skill.** Exports the current conversation to a couple of markdown files in a fixed archive location: a full/verbatim transcript, plus a shorter summary version. Steps: figure out the save location, write the full transcript, write a condensed summary, **update any relevant logs** (the agent's own `memory.md`/`Logs/` folder, and — if this agent is part of a team, [Blueprint #4](../04-Multi-Agent-Interaction/Guide.md) — any shared log it participates in) with what happened in the session, then tell the user where it landed. This means conversation history survives even if the tool's own session storage doesn't, and gives you (or another agent) something readable and searchable later, instead of memory that only lives inside one session.
- **A "close out" skill.** Ends a session deliberately: calls the save skill above (which now includes the log-update step), then closes the terminal window/session it's running in. Worth accepting an optional flag to skip the save step when you just want to close without archiving. Keep this as a *separate* skill from the save skill rather than merging them — if you tend to have multiple agent sessions/tabs open at once, a combined save-and-close skill risks closing more than the one window you meant to, whereas calling save-then-close explicitly (two skills, one after the other) keeps you in control of exactly which window closes.

## Adaptation Notes

- Choose directory-scoped when the agent needs to remember things across unrelated conversations (a status history, a running project list). Choose context-injected when it's a stateless persona dispatched per-task by something else (an orchestrator picking from a roster).
- This pattern isn't Claude Code-specific in spirit — if you're building directly against an API instead of a CLI tool, `CLAUDE.md` maps to a system prompt, `memory.md` maps to retrieved/injected long-term state, and skills map to documented procedures or tool definitions. The *files-as-identity* idea transfers even if the mechanism doesn't.
- Skills can be global (available to every agent) or scoped to one agent's own folder — put a skill in the global location only once more than one agent actually needs it.
- If you're already paying for a subscription plan rather than metered API access, this whole approach effectively rides on a cost you're already covering — worth knowing if you're comparing it against a separately-billed agent framework.

## Gotchas

- **Model inheritance**: if your CLI tool has a global default model setting, an agent without its own explicit model pin can silently inherit it — including an expensive default meant for interactive use, not background agent runs. Always set the model explicitly per agent, don't rely on inheritance.
- **Named agents vs. the tool's own internal subagents**: be explicit in each agent's `CLAUDE.md` about the difference between *these* persistent, named agents you've built (Steve, your NAS agent, etc.) and any built-in "dispatch a subagent for this subtask" feature the CLI tool itself might have. They're different concepts — one is a durable persona with its own memory, the other is usually an ephemeral helper for a single task — and blurring them in the agent's own instructions leads to confusing behavior.
- **Session continuity isn't automatic**: running the CLI tool twice against the same folder does not, by itself, continue the same conversation — that requires deliberately capturing and reusing a session identifier between calls (relevant once you wrap this for Telegram or scripted access — see the bonus sections below).
- **Never hard-code credentials into an agent's identity or memory file.** Route credential lookups through a dedicated secure store instead (see [Blueprint #2](../02-Vaultwarden-Password-Manager/Guide.md) — Vaultwarden).
- A `CLAUDE.md` that's too long or unfocused makes the agent inconsistent — keep identity/personality separate from reference data the agent can go *read* on demand, rather than stuffing everything into one file.
- **The wrapper script only works if its folder is actually on `PATH`.** Most shells need this added explicitly in `.bashrc`/`.profile` (e.g. `export PATH="$HOME/.local/bin:$PATH"`) — check it's there and re-source your shell config, or the command name just won't be found.

---

## Bonus (Optional): Chat With Your Agent via Telegram

Not required — the agent works fine invoked directly. This is for reaching it from your phone as a chat, without opening a terminal. Best suited for quick, one-off messages — checking on something, kicking off an update — rather than extended back-and-forth.

1. Message @BotFather on Telegram, `/newbot`, get a bot token.
2. **Store that token in Vaultwarden ([Blueprint #2](../02-Vaultwarden-Password-Manager/Guide.md)), not in plaintext in your workflow or a `.env` file.** Pull it at runtime with `bw get password "<bot name>"` instead of pasting it directly into n8n credentials or a script — same rule as any other secret this agent touches.
3. Set up something that receives incoming messages and calls your agent with the message text — either:
   - **n8n (default, free)**: a workflow with a Telegram Trigger node → a node that calls your agent (HTTP call to a small local proxy, or directly shelling out to your agent's wrapper command from Step-by-Step #8) → a Send Message node back to Telegram. n8n has its own credential store — even there, prefer pulling the token from Vaultwarden via `bw` in a setup step rather than typing it directly into n8n's UI, so there's one source of truth for it.
   - **Plain script alternative**: a script that calls `getUpdates`, runs your agent's wrapper command with `-p "<message>"` (e.g. `steve -p "<message>"`, or with `--resume <session-id>` for continuity), and posts the reply via `sendMessage` — fetch the token with `bw get password` at the top of the script, not hardcoded.
4. If you want the conversation to persist across messages instead of starting fresh each time, capture the session ID from the first response and pass it back in with `--resume` on the next call.

**Gotcha**: lock down who can talk to your bot. Telegram bots are reachable by anyone who finds the username unless you check the sender's Telegram user ID against an allowlist before acting on a message. The bot token itself is equivalent to a password for that bot — treat a leaked token as a full compromise, not a minor leak. Two layers, both worth doing, proven on the live system: (1) check the numeric Telegram user ID (not the display name — that's spoofable) centrally, wherever your agent-call logic lives, rather than duplicating the check in every workflow; (2) if you're running a local HTTP proxy for the workflow to call, it needs to listen on more than just `localhost` for a Docker-based workflow tool to reach it — which also means it's reachable by anything else on your network unless you firewall it down to just the network segment that actually needs it (a VLAN split like [Blueprint #1](../01-Cloudflare-Tunnel/Guide.md)'s is where this matters most).

## Bonus (Optional): Control Your Agent's Shell From Your Phone

Not required either — this is for full remote-control access to the agent's actual terminal session, not just chat. This is the one worth the extra setup if you want the agent to feel like a familiar chat app on your phone, rather than a one-off command.

1. Claude Code supports `claude --remote-control [name]` — starts an interactive session that can be picked up from the Claude mobile app.
2. Write a small launcher script that opens a terminal and runs your agent's wrapper command (Step-by-Step #8) with `--remote-control` instead of a normal launch, e.g.:
   ```bash
   your-terminal-emulator -- bash -c '/path/to/wrapper/steve --remote-control'
   ```
   The wrapper already handles `cd`-ing into the right folder — the launcher script's only job is opening a real terminal window for it to run in, since remote-control needs an actual interactive session, not just a headless call.

   **In practice, a launcher triggered remotely (SSH/webhook) often can't rely on the wrapper as-is.** The wrapper assumes it's running inside your normal desktop session — a GVFS-mounted folder already up, `DISPLAY`/`WAYLAND_DISPLAY`/`DBUS_SESSION_BUS_ADDRESS` already set by that session. A launcher fired over SSH or a webhook has none of that guaranteed. If the wrapper hangs or fails when triggered this way, have the launcher script `cd` into the agent's folder via a boot-time mount path instead (e.g. an `/etc/fstab` mount, not a GVFS/session mount) and set `DISPLAY`, `WAYLAND_DISPLAY`, and `DBUS_SESSION_BUS_ADDRESS` explicitly, rather than assuming they're inherited.
3. Trigger that script remotely — a webhook (n8n again, or anything that can SSH in and run a command) that fires the script on your machine when you want to start a session from your phone.
4. Open the Claude mobile app → the session shows up for pickup.

**Gotcha**: this hands over a real interactive shell, not just chat — restrict who/what can trigger the launcher the same way you'd restrict any remote command execution.
