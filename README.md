# Homelab AI Blueprints

**What this is:** Self-contained blueprints for building a self-hosted, AI-agent-operated home server — the kind of setup where a Claude Code-style AI CLI assistant does the actual configuration work with you, not just answers questions about it. Each blueprint is written twice:

- **`Guide.md`** — plain, step-by-step, for a person doing it themselves.
- **`AI-Brief.md`** — written to brief an AI assistant directly: enough context that it can read the file, ask you a few clarifying questions about your specific setup, and then do the task for you.

## How to use it with an AI

Point your AI assistant at a blueprint's `AI-Brief.md` and tell it to follow it. The brief tells the AI what questions to ask you before starting and what "done" looks like, so you don't have to translate the instructions yourself. If you'd rather do it by hand, read `Guide.md` instead — same material, written for a person with the full reasoning included.

## Blueprints (in dependency order)

Build roughly in this order — later blueprints assume earlier ones exist.

1. **[Cloudflare Tunnel](01-Cloudflare-Tunnel/Guide.md)** — expose self-hosted services on public subdomains without opening router ports.
2. **[Vaultwarden + bw CLI](02-Vaultwarden-Password-Manager/Guide.md)** — a self-hosted password manager your scripts and AI agents can pull credentials from programmatically, instead of hardcoding them.
3. **[Agent Creation](03-Agent-Creation/Guide.md)** — build a persistent, directory-scoped AI agent with its own identity, memory, and a CLI wrapper you can call by name from anywhere. Includes optional Telegram and phone remote-control bonuses.
4. **[Multi-Agent Interaction](04-Multi-Agent-Interaction/Guide.md)** — coordinate multiple agents, either through a director that dispatches work or a shared-knowledge-base pattern for independent peers.
5. **[Scheduled Self-Reporting](05-Scheduled-Self-Reporting/Guide.md)** — agents that check themselves (and each other) on a cadence, plus an independent-hardware watchdog backstop for when the primary machine itself is down.
6. **[Audiobook Pipeline](06-Audiobook-Pipeline/Guide.md)** — convert purchased audiobooks to DRM-free files and serve them to family without sharing your account credentials.
7. **[Spotify Playlist Pipeline](07-Spotify-Playlist-Pipeline/Guide.md)** — turn a Spotify playlist into a self-hosted, deduplicated music library with a real Plex playlist.
8. **[Email Archiving](08-Email-Archiving/Guide.md)** — pull your mailbox down locally, archive it year-by-year, and free up provider storage.
9. **[SMS Archive + Search Agent](09-SMS-Archive-And-Search/Guide.md)** — build a searchable, private archive of your text message history that an AI agent can query on request.

Also included:
- **[Services Overview](00-Services-Overview.md)** — a fast-lookup reference for every tool/service mentioned across the blueprints (what it is, why a project here needs it, verified download link) — not setup steps, just background.
- **[Skills Catalog](00-Skills-Catalog/Guide.md)** — reusable, generic AI-agent skills (save/close-out a conversation, self-report on a schedule, log changes for other agents) that aren't tied to any one blueprint's service.

## Prerequisites

- A Linux machine (these blueprints were built and verified on Ubuntu; other distros should work with minor adjustments).
- Docker, for blueprints that run containerized services.
- A domain name, for Blueprint 1 (Cloudflare Tunnel) and anything built on top of it.
- An AI CLI assistant (Claude Code or equivalent) if you want the `AI-Brief.md` files to actually execute the work for you, rather than following `Guide.md` by hand.

## A note on security

These guides involve credentials, tunnels, and remote access. Never commit secrets (API keys, tokens, passwords) to a repository — including one you fork from this. The blueprints' own credential rules exist for a reason: route secrets through a password manager (Blueprint 2's Vaultwarden pattern) rather than plaintext config files, and read Blueprint 5's Gotcha on the one deliberate exception (a watchdog on separate hardware, where a password-manager dependency would defeat its purpose) before assuming that rule is absolute.

See [CHANGELOG.md](CHANGELOG.md) for what's changed since initial publish.

## Freshness

The blueprints below were verified against a live deployment and current official documentation as of **2026-07-29**. Anything added after that date carries its own verification date. Version numbers and pricing mentioned throughout (Cloudflare Registrar pricing in Blueprint 1, Vaultwarden's websocket behavior since v1.29.0 in Blueprint 2, OpenAudible pricing in Blueprint 6, spotDL 4.5.x in Blueprint 7) were accurate as of that date — check the linked official sources if you're reading this well after.

## License

[CC BY 4.0](LICENSE) — use, adapt, and share freely, with attribution.
