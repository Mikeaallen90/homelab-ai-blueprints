# Blueprint #5 — Scheduled Self-Reporting Agents (+ Independent Watchdog Backstop) (AI Brief)

## Purpose

Build a director agent that periodically directs each service-specific agent to check itself (health, and separately, available updates) and report back, rolling results up into a single alert only when something needs attention. See `Guide.md` for the full human-readable version — this file is for an AI assistant executing the setup on the user's behalf.

## Questions to Ask Before Starting

1. How many agents exist already, and do they have a director/coordinator, or are they standalone ([Blueprint #3](../03-Agent-Creation/AI-Brief.md)/[#4](../04-Multi-Agent-Interaction/AI-Brief.md) status)?
2. What does "healthy" mean for each service — what should its agent actually check (not just up/down, but the real domain-specific signal)?
3. What scheduler is available — n8n, cron, systemd timer? What cadence for health checks (default every 6h) vs. update checks (default weekly)?
4. Where should each agent log its results — existing log files/notes, or something new to set up?
5. Is a messaging bot already wired up for alerts ([Blueprint #3](../03-Agent-Creation/AI-Brief.md)), or does that need doing first?
6. Should updates auto-apply for patch/minor versions, or should this job be report-only?

## What "Done" Looks Like

- A scheduled health-check run fans out to every service agent, each reports its own status, and the director sends exactly one rolled-up alert only if something needed attention (not one per agent).
- A separate scheduled update-check run does the same fan-out for update-checking, auto-applies patch/minor updates where enabled, and produces a full written report for any major version bump.
- A deliberate test — manually break one service — gets caught on the next scheduled run and rolled into a single alert.
- Each agent's own log shows both healthy and unhealthy checks, not just failures.

## Key Points to Carry Into the Build

- A leader/director agent that delegates to specialists is what keeps multi-agent setups from becoming confusing about who's responsible for what — don't skip this once there's more than one or two agents.
- Most self-hosted services don't update themselves — the update-check job exists specifically to remove that manual, per-service chore.
- Roll multiple agent results into a single alert per run — never send one alert per agent for the same scheduled pass.
- This job assumes the primary machine is up and running. It is not a substitute for the independent watchdog backstop below, which exists for the case where the whole machine is down and nothing here can even run.

---

## Extra: Independent Watchdog Backstop (AI Brief)

### Purpose

Build a monitoring agent on a second, independent device that watches a primary machine's devices/services, attempts automatic recovery, and alerts the user via chat on both failure and recovery. This is a backstop for the primary self-reporting content above — it only makes sense once you've read that section, since its whole point is covering the case where self-reporting can't run at all (the primary machine is fully down).

### Questions to Ask Before Starting

1. What's the second device — already have one repurposed (old laptop/PC), or provisioning something new (Raspberry Pi, cloud VM)? What OS is on it?
2. What's the primary machine's IP/hostname, and what services on it need watching (name + port or systemd service name for each)?
3. Do you have (or want) SSH key access from the watchdog to the primary machine? To the router, for port-cycling?
4. Do you have a messaging bot ready for alerts, or does [Blueprint #3](../03-Agent-Creation/AI-Brief.md)'s Telegram bonus need to be done first?
5. Do you have a managed switch or smart plug available for hard power-cycling, or should the power-cycle tier be skipped?
6. How often should it check (default: every 5 minutes)? Any services that need more tolerant retry/delay settings than others (e.g., something known to blip briefly and self-heal)?
7. Does anything else (an update script, another agent) need to be able to trigger maintenance mode itself, or is it user-only?

### What "Done" Looks Like

- Systemd timer (or cron) fires the monitor script on schedule, confirmed via logs.
- A deliberate test (stop a real service) produces: immediate "down, attempting X" alert → automatic recovery attempt → a follow-up alert reporting the outcome.
- A deliberate test of the primary machine's network being unreachable produces a device-down alert and confirms the watchdog itself keeps running (proves independence).
- Maintenance-mode start/stop/status all work, and the flag auto-expires if left on.
- Logs capture both successful and failed checks, not failures only.

### Key Points to Carry Into the Build

- Confirm-retry logic before alerting is not optional — it's what prevents transient blips from becoming false alarms that train the user to ignore real ones.
- Alerts must state what corrective action is being taken, not just that something is down.
- The "still down after auto-fix, escalate to a human" step should be a distinct, clearly labeled step in the script — not folded into the initial alert-on-down step.
- Maintenance mode should be callable by other scripts/agents, not just run manually by the user — this is how a user-initiated update, or another agent's own automation, avoids triggering false-down alerts during planned work.
- Verify the actual hardware/network independence before calling the build done — if the watchdog shares a power source or network path with what it's monitoring, the design goal isn't actually met.
- Don't put bot tokens in the config file in plaintext, and don't route them through Vaultwarden either — that reintroduces a dependency on the primary machine this watchdog is meant to survive without. Use a local encrypted-at-rest store instead (`systemd-creds` is the built-in option); ask the user if they want help wiring that up rather than defaulting to plaintext.
