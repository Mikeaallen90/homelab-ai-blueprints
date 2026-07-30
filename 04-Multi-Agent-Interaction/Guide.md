# Multi-Agent Interaction — Team Collaboration + Shared Knowledge Base

## Why This Exists

When you have multiple agents, you'll want them to remember things. You may want to talk to one agent about something you did with another, want your agents to be experts at what you task them with, or even give them limited access. I have teams, and the skills and setup I have let me log, track, and keep information across agents.

## What It Does

Covers the two things that come up as soon as you have more than one agent: coordinating several specialists toward one shared goal or deliverable (a novel, a system audit, anything with distinct phases of work), and giving separate, independent agents a lightweight shared space to stay aware of each other without duplicating each other's knowledge. Set both up from the start rather than bolting them on later — retrofitting a shared knowledge base onto agents that have already been running independently for a while is more work than building it in alongside them.

## What You'll Need

- At least two agents built via [Blueprint #3](../03-Agent-Creation/Guide.md) (or one agent built with multiple distinct "modes" it can be told to act in — see Adaptation Notes)
- A director role — either a dedicated agent, or you the human, coordinating dispatches (only needed for the director-dispatch pattern, below)
- A way to invoke a specialist for one task and get a result back (this can be your CLI tool's own built-in subagent-dispatch feature, if it has one, or manually calling each named agent's wrapper command from #3)
- A shared folder any of your agents can read and write to (for the shared-knowledge pieces below)

## How It Works

**Specialists are stateless per dispatch.** Each call to a specialist is a fresh session with no memory of any earlier dispatch — it only knows what's in the one prompt it's given. The director is what carries continuity across the whole task; specialists don't need to, and shouldn't be expected to, remember prior rounds.

**The director holds the plan, not the work.** It breaks the overall goal into focused tasks, decides which specialist handles each one, passes each specialist exactly the context it needs for that task (nothing more), and integrates results as they come back.

**Review stays separate from drafting, always.** If one specialist's job is to produce something (write a scene, make a change) and another specialist's job is to check it (continuity, correctness, quality), never let the same dispatch do both, and never merge a review dispatch into the drafting dispatch that produced the thing being reviewed — even when they'd otherwise be efficient to combine. A drafting agent checking its own output tends to confirm its own assumptions.

**Fewer, broader specialists beats many narrow ones.** A large roster of single-purpose agents sounds clean but multiplies dispatch overhead — more separate calls for the director to make, more usage/token cost, more coordination surface. Grouping related narrow jobs into one specialist with a few named "modes" (it's told which mode to act in per dispatch) cuts the dispatch count substantially without losing the separation that actually matters (review vs. drafting).

**Give each specialist only the access it needs.** "Expert at what you task it with" also means *limited* to what that task needs — a specialist that only touches one domain should only be able to see/change files in that domain, not the whole system. Directory-scoping an agent ([Blueprint #3](../03-Agent-Creation/Guide.md)) naturally limits what it sees just by which folder it runs from; where your tool supports finer-grained permissions, use those too. This isn't just tidiness — it keeps a specialist that's given a bad task, or misbehaves, from having a bigger blast radius than its actual job requires.

## Two Shapes of "Separate but Connected"

The director/specialist pattern above isn't the only shape multi-agent coordination takes. Worth knowing both, since which one fits depends on the type of coordination you need — and most real setups end up using both at once, for different parts of the system.

**Director-dispatch** — one agent holds the overall plan and actively hands out tasks. Specialists don't need to know about each other; the director is the single point of coordination. Fits well when there's one deliverable being built in phases.

> *Example:* A "code review team." A director breaks a pull request into tasks: a Style specialist checks formatting/conventions, a Logic specialist traces through the actual change for bugs, and — kept strictly separate from both — a Verifier specialist independently confirms the proposed fixes actually work, never folded into the same dispatch as the specialist that proposed them. The director collects all three results and writes the final review comment.

**Peer/shared-knowledge pattern** — no central dispatcher. Each agent owns a separate domain and keeps its own knowledge to itself. What connects them is a shared space they all read and occasionally write to — lightweight by design, not a duplicate of anyone's full memory.

## Building the Shared Knowledge Base

This is the concrete version of the peer pattern above — a real, small set of files any of your agents can reach, that keeps independent agents aware of each other without merging their memories together.

1. **Create one shared folder** all your agents can read and write to (a vault folder, a shared drive, anything that's not any single agent's private directory).
2. **Start with an introductions file.** The first time a new agent is added, it writes a short entry: its name, what it does, what it can do alone vs. together with others, and any personality it has. This does double duty — it's useful documentation, and gives each agent a quick way to "meet" the others by reading past entries before acting.
3. **Add a routing reference** — a table of which agent owns which domain, where to find that agent's files, and when a task should be handed off instead of handled locally:

   | Agent | Owns | Escalate When |
   |---|---|---|
   | Storage agent | Drives, backups, shares | Task needs network changes |
   | Network agent | Router, VLANs, DNS | Task needs a device it doesn't manage |
   | Media agent | Streaming server, libraries | Storage or network issue is the actual cause |

4. **Add a handoff template** for the moments a task genuinely needs to move from one agent to another — not just a quick note, but enough context that the receiving agent doesn't have to reconstruct everything from scratch:

   ```markdown
   ## Handoff — [Date]

   **From**: [Agent name]
   **To**: [Agent name]
   **Urgency**: [low / normal / high]

   ### Request
   [What was asked, verbatim or close to it]

   ### Goal
   [What a successful outcome looks like]

   ### Context Already Checked
   [Files, docs, other agents already consulted before this handoff]

   ### Checks Already Run
   [What was verified, tested, or ruled out — be specific]

   ### Files / Services Touched
   [Anything already modified or that the receiving agent should be aware of]

   ### Recent Relevant Changes
   [Anything logged in the shared change log recently that might matter here]

   ### Constraints
   [Things not to touch, scope limits, relevant preferences]

   ### Requested Output
   [What the receiving agent should return]

   ### Unresolved Questions
   [Anything unclear the receiving agent may need to decide or ask about]
   ```

5. **Add a running change log** — one shared place where any agent logs a significant change it made, so others can check "what's changed recently" before answering, without needing each other's full history. Real shape it can take: *"Storage agent expanded the media volume — Media agent should note more space is now available for new libraries."*
6. **Tell every agent, in its own identity file, to actually use this folder** — read the change log (and any relevant routing/handoff entries) before answering a status question, and write a change-log entry after doing anything another agent might need to know about. The shared folder does nothing on its own; agents have to be explicitly instructed to check it, or it just sits there unread.

This fits well when there's no single deliverable — just a set of independent domains that occasionally need to hand off to each other or stay aware of each other's recent changes, more like a company's departments than a project team. Adding a new domain just means adding a row to the routing table, not restructuring a dispatch flow. And it composes with director-dispatch: a whole "team" from the pattern above can be one row in a larger routing table, participating in the wider shared knowledge base as a single unit.

## Step-by-Step Setup

**Director-dispatch (one shared deliverable):**
1. Define the shared goal/domain this team exists for (a creative project, a recurring system task, anything with distinct phases).
2. List the distinct *kinds* of work involved, then group related ones into a handful of specialists rather than one agent per narrow task — each specialist can have multiple named modes instead of needing a whole separate agent per mode.
3. Build each specialist per [Blueprint #3](../03-Agent-Creation/Guide.md) (identity, and memory if it needs to retain anything between separate projects, not between dispatches within one project).
4. Keep at least one specialist whose only job is independent review/checking — never fold its role into a drafting specialist's dispatch.
5. Build or designate the director: either a dedicated agent whose job is purely to plan and dispatch (not to do specialist work itself), or a documented process for a human to do the same coordination manually.
6. Write the dispatch order/routing logic down somewhere the director can reference — which specialist handles which kind of task, and the rule that consecutive same-specialist steps can be merged into one dispatch, but a review dispatch never merges into the drafting dispatch it's reviewing.
7. Scope each specialist's actual access (folders, tools) to what its domain needs, not the whole system.

**Shared knowledge base (independent domains staying connected):**
8. Create the shared folder and the introductions file (Building the Shared Knowledge Base, steps 1–2).
9. Add the routing reference and handoff template (steps 3–4).
10. Add the change log (step 5).
11. Update every participating agent's identity file to reference the shared folder — when to read it, when to write to it (step 6). Do this for every agent going forward, not just the ones that exist today — it's much easier to build this in from an agent's first day than to retrofit it after the agent's been running independently for a while.

**Both patterns:**
12. Add a closeout step for wrapping up a working session (see [Blueprint #3](../03-Agent-Creation/Guide.md)'s "Useful Skills to Build First" — the save/close-out pattern applies here too, and should log to the shared change log as part of that closeout whenever the session touched something other agents should know about).

## Adaptation Notes

- This isn't creative-writing-specific — the same shape applies to any multi-phase task: a "specialist per domain, one integrator, review kept separate" pattern works for a system-audit team, a document-review pipeline, anything with distinct kinds of work and a need for an independent check.
- If your CLI tool has a native subagent-dispatch feature, you can use that instead of manually invoking separate named agents' wrapper commands — same pattern, different mechanism for the "call a specialist, get a result back" step.
- Start with more specialists than you think you need if you're unsure, then consolidate narrow ones together once you see which jobs are actually related — going from many to few is easier than the reverse, since you're just merging identity files, not losing any capability.
- The shared knowledge base doesn't need every file from day one — the introductions file and change log are worth having immediately; the routing reference and handoff template can be added once you actually have enough agents that handoffs are a real, recurring thing.
- Director-dispatch and the shared knowledge base aren't mutually exclusive — a whole team can participate in a wider shared knowledge base as a single row in the routing table.

## Gotchas

- **Don't assume a specialist remembers anything from an earlier dispatch.** If it needs prior context, the director has to explicitly include it in the new prompt — nothing carries over automatically.
- **Never let a drafting specialist also be its own reviewer**, even when merging the two dispatches would save a call. This is the one rule worth protecting even when everything else gets consolidated for efficiency.
- Too many narrow specialists is a real cost, not just clutter — each one is a separate dispatch the director has to make, which adds up in both time and usage against any token/rate limits you're working within.
- A director that does specialist-level work itself (instead of only planning and dispatching) tends to blur the whole pattern back into "one agent doing everything," losing the benefit of focused, independently-reviewed specialists.
- A specialist with broader file/tool access than its job requires increases the damage a bad task or a mistake can do — keep access scoped to the domain, mirroring the directory-scoping principle from [Blueprint #3](../03-Agent-Creation/Guide.md).
- **A shared knowledge base that no agent is instructed to read is just a folder.** The value only shows up once agents actually check it before answering and write to it after acting — that has to be spelled out in each agent's own identity file, not assumed.
- **Even instructed, agents still forget to log — especially in headless or one-off runs.** Identity-file instructions degrade over a long session and aren't enforced by anything. If your platform supports lifecycle hooks (e.g. Claude Code's Stop hook, which runs a script when a session tries to finish), the durable fix is mechanical, not textual: a hook script that scans the ending session's transcript for shared-state changes — mutating service/system commands (including ones wrapped in `ssh` to other machines), or file writes outside the agent's own folder — and blocks the session from finishing until the logging skill has run or the agent states an explicit one-line waiver (e.g. `NO-LOG: <reason>`). Keep it *soft*: waiver allowed, block at most once per stop, fail open on any script error so it can never trap a session. And exempt the agent's own upkeep (its memory/log files) and its normal deliverables (saved conversations, content it produces for the user) so the hook doesn't cry wolf — false positives teach agents to game the waiver instead of log honestly. Enforcement then lives in the harness rather than the model's memory, which is the only place "always do X" is actually guaranteed.
- **The change log will grow forever if nothing ever prunes it.** Deciding a pruning rule isn't enough — a rule that lives only in a doc gets forgotten (ours sat unenforced for two months). Build rotation into the logging skill itself as a step: e.g. at each month boundary, move prior-month rows *verbatim* into a per-month archive file (`ChangeLogArchive/{YYYY-MM}.md`), always keeping the most recent N rows live so early-month sessions still see recent context, and point at the archive folder from the live table's intro line. A rule that runs every time the log is written to can't be forgotten.
