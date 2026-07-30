# Multi-Agent Interaction — Team Collaboration + Shared Knowledge Base — AI Brief

Read this, then ask the user the questions below before doing anything. Goal: stand up multiple specialist agents (already built or to be built per [Blueprint #3](../03-Agent-Creation/AI-Brief.md)) coordinating toward one shared goal, and/or connect independent agents through a shared knowledge base — pick the right shape(s) first, build both from the start where relevant rather than only the one asked about, since retrofitting the shared knowledge base later is more work than building it in from an agent's first day.

## Context

Two coordination shapes exist, and they solve different problems, and most real setups use both for different parts of the system:
- **Director-dispatch**: one agent plans and hands out focused tasks to specialists, integrates results. Fits one deliverable built in phases.
- **Shared knowledge base**: no central dispatcher — each agent owns a separate domain and keeps its own knowledge to itself, connected by a shared folder containing: an introductions file (each agent's self-written intro), a routing reference (who owns what, when to hand off), a handoff template (structured context transfer between agents), and a running change log (recent developments any agent can check).

Core rules for director-dispatch regardless of domain: every specialist dispatch is a stateless, fresh session — the director must include all needed context in each prompt, nothing carries over automatically. Review/checking must never be the same dispatch as the drafting/work it's reviewing, even when combining would be more efficient. Prefer fewer, broader specialists (with named "modes") over many single-purpose ones, to control dispatch overhead. Each specialist's actual file/tool access should be scoped to its domain, not the whole system.

Core rule for the shared knowledge base: it does nothing unless every participating agent's identity file explicitly tells it to read the shared folder before answering and write to it after doing anything other agents might need to know about. A shared folder nobody's instructed to check is just an unread folder.

## Questions to ask before starting

1. Is there one shared deliverable being built in phases (→ director-dispatch), a set of independent domains that occasionally need to coordinate (→ shared knowledge base), or both?
2. What are the distinct kinds of work involved? Group related ones — don't default to one agent per narrow task.
3. Which specialist (if director-dispatch) is the independent reviewer/checker? Confirm it's never the same as any drafting specialist.
4. For each specialist/agent, what access does its job actually require (files, tools, folders)? Don't grant more than that by default.
5. Does a shared knowledge base already exist to plug into, or does one need creating? If creating one and there are already-existing agents (not brand new), flag that retrofitting them (updating their identity files to reference it) still needs doing — don't silently skip existing agents just because they predate this setup.

## Steps to execute

**Director-dispatch:**
1. Build/confirm each specialist per [Blueprint #3](../03-Agent-Creation/AI-Brief.md), designate or build the director (planning/dispatching only, not doing specialist work itself), and write down the routing logic — which specialist handles which kind of task, and the never-merge-review-into-drafting rule.
2. Scope each specialist's actual access to its domain.

**Shared knowledge base:**
3. Create the shared folder (any location every intended agent can read/write).
4. Create the introductions file; have each existing agent (and every new one going forward) write a short self-introduction into it.
5. Create the routing reference (table: agent, domain owned, when to escalate) and the handoff template (see Guide.md for the exact template — From/To/Urgency/Request/Goal/Context Already Checked/Checks Already Run/Files Services Touched/Recent Relevant Changes/Constraints/Requested Output/Unresolved Questions).
6. Create the running change log.
7. Update every participating agent's identity file (`CLAUDE.md` or equivalent) to explicitly instruct it to read the change log before answering status questions and write an entry after making a change others should know about. Do this for agents that already exist, not just new ones.

**Both:**
8. Wire the closeout skill ([Blueprint #3](../03-Agent-Creation/AI-Brief.md)'s save/close-out pattern) to also log to the shared change log as part of closing out a session, when the session touched something other agents should know about.

## Watch for

- A director that starts doing specialist-level work itself instead of only dispatching — this erodes the whole pattern back into one agent doing everything.
- Any temptation to merge a review dispatch into the drafting dispatch it's reviewing "just this once" — don't.
- A specialist granted broad access "to be safe" — access should match the job, not be maximized preemptively.
- Confirm dispatches genuinely don't rely on a specialist recalling an earlier one; if continuity is needed, it must be explicitly passed in each time.
- Building the shared folder but forgetting to actually instruct agents to use it — the folder existing is not the same as it being read.
- Existing agents left out of the shared-knowledge rollout because the user only mentioned "the new one" — ask explicitly whether existing agents should be updated too.
- Instructions alone won't guarantee logging — agents forget, especially headless/one-off runs. If the platform has lifecycle hooks (e.g. Claude Code's Stop hook), offer the mechanical enforcement pattern from Guide.md's Gotchas: a hook that blocks session end when shared state changed but the logging skill never ran (soft-block, explicit waiver allowed, fail-open, own-folder/deliverable writes exempt). Likewise build change-log rotation into the logging skill itself (month-boundary archive, keep last N rows live) rather than documenting a pruning rule nothing enforces.

## Done when

- Each specialist (or peer agent) exists per [Blueprint #3](../03-Agent-Creation/AI-Brief.md) and responds correctly for its scoped domain.
- Director-dispatch: a full task cycle — director dispatches, specialist(s) produce work, an independent specialist reviews it, director integrates — completes end to end with the review never merged into drafting.
- Shared knowledge base: the folder exists with at minimum an introductions file and a change log; every participating agent's identity file references it; at least one real change-log entry from one agent is checkable by another.
- Closeout for a session logs to the shared change log when relevant, not just the individual agent's own conversation archive.
