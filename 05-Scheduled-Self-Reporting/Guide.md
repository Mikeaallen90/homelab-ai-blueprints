# Blueprint #5 — Scheduled Self-Reporting Agents (+ Independent Watchdog Backstop)

## 0. Why This Exists

Once you have more than one agent running, it's easy to lose track of who's supposed to be doing what, or to forget to check in on each one individually. Having a leader — someone you talk to, who delegates the actual checking to the right specialist — makes the whole thing simpler, instead of you having to remember to ask five different agents if they're okay.

Checking for updates deserves its own setup too, because a lot of services don't update themselves — without something checking on a schedule, keeping things current becomes a manual chore, service by service, that you have to remember to do yourself.

Services, security, and network monitoring is something everyone running their own hardware needs — this blueprint and its watchdog backstop below (see "Extra") are two halves of that.

## 1. What It Does

A coordinator agent periodically directs each service-specific agent to check itself and report back — status now, and separately, whether an update's available. Each agent uses its own domain knowledge to assess its own service, rather than a generic external probe checking the same shallow thing for everything. Most self-hosted services don't update themselves, so without something checking on a schedule, staying current becomes a manual chore per service — this handles that automatically wherever it safely can.

## 2. What You'll Need

- At least one agent already built ([Blueprint #3](../03-Agent-Creation/Guide.md)) — ideally several, one per service you're tracking ([Blueprint #4](../04-Multi-Agent-Interaction/Guide.md) if you want them coordinated by a director rather than run standalone)
- A scheduler — n8n, cron, or a systemd timer — to fire the check on a cadence
- A log destination per agent (a file, a note, a database row) to write results to
- A messaging bot for alerts when something can't recover on its own ([Blueprint #3](../03-Agent-Creation/Guide.md)'s Telegram bonus)

## 3. How It Works

A director agent wakes up on schedule and fans out to each service agent in parallel (batched in small groups if you have many, to avoid hammering everything at once). Each agent:

1. Checks its own service using whatever it actually knows — not a generic ping/port check, but real domain checks (is the library loading, is the container healthy, is the expected data present)
2. Attempts its own recovery if it can (restart itself, clear a stuck state)
3. Logs the result (UP / DOWN / RECOVERED) to its own log
4. Reports back to the director

The director collects all results and, only if something couldn't recover on its own, sends one alert — not one per agent, a single rollup so you're not woken by five separate pings for one root cause. A second, separate scheduled job (weekly, say) does the same fan-out but for update-checking instead of health-checking: each agent checks its own service for available updates, applies patch/minor versions automatically, and writes a fuller report if a major version bump needs a human decision.

## 4. Step-by-Step Setup

1. Make sure you have a director agent and at least one service agent to report to it ([Blueprints #3](../03-Agent-Creation/Guide.md)/[#4](../04-Multi-Agent-Interaction/Guide.md)).
2. Give each service agent its own "check yourself" instructions in its identity file — what healthy looks like for its specific service, and what to do if it isn't.
3. Write the director's dispatch script/skill: fan out to agents in parallel batches, collect results, roll up into one alert only if something needs attention.
4. Add a log-write step to each agent's check — its own status log, append-only.
5. Wire the director's alert to your messaging bot.
6. Schedule the health-check job (e.g., every 6 hours) via n8n/cron/systemd timer.
7. Repeat steps 3–6 for a separate update-check job on a slower cadence (e.g., weekly) — same fan-out shape, different question being asked, with an escalation path (full written report) for anything needing a manual decision.
8. Test: manually break one service, confirm it's caught on the next scheduled run and rolled into one alert, not several.

## 5. Adaptation Notes

- Only have one or two agents? Skip the "director fans out" complexity — a single script checking itself and logging is still worth having, you just lose the multi-agent rollup.
- No auto-apply appetite for updates? Make the update-check job report-only — still worth running, just without the "apply automatically" step.
- This same scheduled-check-and-notify shape doesn't have to point at your own services — [Blueprint #3](../03-Agent-Creation/Guide.md)'s "Repo Update Checker" bonus reuses it to watch an external GitHub repo (specifically, the blueprints repo these guides come from) instead.

## 6. Gotchas

- Don't let this duplicate the watchdog backstop below — this job only runs if the machine itself is up. Skip the backstop entirely and you have no way to know when the whole thing goes dark.
- A single rolled-up alert per run (not one per agent) matters — the whole point of a coordinator is sparing you alert spam when five things fail from the same root cause.

---

## Extra: Independent Watchdog Backstop (optional — run this on separate hardware if possible)

Scheduled self-reporting above only works if the agents doing the reporting can actually run — if the whole primary machine goes down, there's nothing left to schedule and nothing gets reported. Silence, not an alert. This backstop closes that gap: a second, independent device that watches the primary machine from the outside and can tell you it's down even when nothing on the primary machine itself is able to.

### Why This Exists

Most of the services I run locally live on a PC that stays on and running — even a free service still needs hardware to host it. But hardware and services both have problems: things go down, glitches happen. I wanted something watching that, catching it, and telling me what's happening — which is actually the main reason I use Telegram notifications for these agents at all: so an agent can tell me something happened *and what's being done about it*, not just that it happened.

My monitoring agent runs on an old laptop I wiped and put Linux on, kept deliberately separate from my main PC — if the PC itself goes down, I still want to hear about it. It watches my services, restarts one if it can, notifies me, and logs everything. I (or another agent) can put it into maintenance mode when I need to do updates or manual work, so it doesn't cry wolf while I'm the one causing the outage.

I think everyone running anything on their own hardware should have something like this. The last thing you want is to wake up and find out something's been down all night and nobody told you.

### What It Does

Runs on its own hardware, checks your main system's pulse every few minutes, attempts automatic recovery on failure, and alerts you via chat — both when something breaks and when it comes back — without you having to notice or go looking.

### What You'll Need

- A **second always-on device**, physically/electrically separate from the machine it watches (old PC, Raspberry Pi, spare laptop, even a cheap cloud VM). The point is an independent failure domain: different power circuit and ideally different network path than what it's monitoring. Cheapest real option: wipe an old laptop or PC you're not using anymore and put a lightweight Linux distro on it — no need to buy anything new.
- SSH key access from the watchdog to the main machine (and to your router, if you want port-level power-cycling)
- A messaging bot for alerts (see [Blueprint #3](../03-Agent-Creation/Guide.md)'s Telegram bonus section)
- Optional: a managed switch or smart plug you can control remotely, for hard power-cycling when SSH itself is unreachable

This assumes you already have a primary machine running the self-hosted services you care about (see the other blueprints). The hardware requirement here is specifically the *second*, independent device — not the machine doing the actual work.

### How It Works

A systemd timer (or cron) fires a monitor script every N minutes. The script runs two tiers of checks:

1. **Device-level** — plain ping. Is the box even reachable?
2. **Service-level** — port check (does something answer on the expected port?) or, if you have SSH access, a `systemctl is-active` check for more precision than a port check alone gives you.

On failure, it doesn't panic-alert immediately — it re-checks a few times with a short delay first (confirm-retries), since transient blips (a reboot mid-update, a momentary network hiccup) are common and false alarms train you to ignore real ones. Only after a failure is *confirmed* does it:

1. Send an immediate alert via chat bot — and the alert should say what's being done, not just that something's down. "X is down, attempting restart" tells you far more than "X is down."
2. Attempt a tiered automatic fix — restart the specific service first (cheapest, fastest), fall back to power-cycling the network port or the whole device only if that fails
3. Poll for recovery for a bounded window
4. Send a second alert either way — "recovered" or "still down, manual intervention needed"

A **maintenance-mode flag** lets you suppress the whole scan for a set window during planned work (an update, a reboot you triggered on purpose). It can be flipped on either by you directly, or by another agent/automation as part of its own process — e.g., an update script can start maintenance mode itself right before it begins work, rather than you having to remember to do it by hand. It auto-expires as a safety net so a forgotten flag can't silently disable monitoring forever.

### Step-by-Step Setup

1. Provision the second device, get it on the network, confirm it can reach the main machine.
2. Set up SSH key auth: watchdog → main machine (and → router, if using port-cycling).
3. Write a config file: IPs/hostnames to watch, bot tokens (see Gotchas — don't leave these in plaintext), retry/delay tuning.
4. Write the monitor script: device checks (ping) → service checks (port/systemctl) → confirm-retry logic → alert → corrective action → recovery poll → alert. (Step 6 below adds a maintenance-mode check to the front of this same script — not a separate script.)
5. Wire up the messaging bot for alerts (see [Blueprint #3](../03-Agent-Creation/Guide.md)).
6. Add a maintenance-mode flag file plus a start/stop/status helper script, and check it first thing in the monitor script (skip the whole scan if active). Make sure other scripts/agents can call the "start" action too, not just you manually.
7. Create the systemd timer + oneshot service (or a cron entry) to run the monitor script every 5 minutes.
8. Test deliberately: stop a real service and confirm alert → auto-fix → recovery alert all fire correctly. Then pull the network cable on the main machine and confirm the *watchdog* survives and alerts (this is the test that actually validates the "independent" part).

### Adaptation Notes

- No managed switch/smart plug? Drop the power-cycle tier — restart-service and alert-only still work fine, you just lose the "make the port itself flap" fallback.
- No SSH access to the main machine (e.g., watching a device you don't control)? Ping + port checks alone still catch most outages, you just lose the `systemctl` precision tier and the ability to auto-restart anything.
- Confirm-retry counts/delays should be tuned per-service — a flaky-but-fine service (like a tunnel doing a routine edge rotation) needs a more tolerant window than something that's either up or truly down.

### Gotchas

- If the watchdog and the thing it watches share a power strip, a router, or a network switch, you haven't actually built independence — a single outage takes both down and you get silence, not an alert.
- Bot tokens in that config file (Step 3) sit in plaintext, which cuts against the Vaultwarden-based credential rule from [Blueprints #2](../02-Vaultwarden-Password-Manager/Guide.md)/[#3](../03-Agent-Creation/Guide.md). That rule assumes the primary machine's password manager is reachable — but the whole point of this watchdog is surviving the primary machine being down, so depending on it for a credential defeats the design. If you want tokens off disk in plaintext, a local encrypted-at-rest store is the right middle ground: `systemd-creds` (built into systemd 250+, no extra software needed) encrypts a secret to a file only the local machine's own key can decrypt — a systemd service loads it automatically via `LoadCredentialEncrypted=`, and anything else on the box can decrypt it on demand through a narrowly-scoped `sudo` rule. Two things that will trip you up: the host key file must keep systemd's own strict default permissions (`root:root`, `0400`) or it refuses to encrypt/decrypt at all, and the credential file's name must exactly match the name embedded at encryption time (no arbitrary file extension).
- The escalation step ("still down after auto-fix, notify a human/coordinator") is a distinct concern from raw monitoring — worth keeping as a separate, clearly-labeled step rather than blurring it into the alert-on-down step, so you can tell "detected + fixed" apart from "detected + gave up."
- Log everything the script does, even successful checks, not just failures — when something *does* go wrong you'll want the timeline of what was healthy right before it.
