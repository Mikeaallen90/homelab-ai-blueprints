# Vaultwarden — AI Brief

Read this, then ask the user the questions below before doing anything. Goal: stand up a self-hosted, Bitwarden-compatible password manager with working CLI access for scripts/agents.

## Context

Vaultwarden is an open-source, self-hosted reimplementation of the Bitwarden server. It speaks the same protocol as official Bitwarden clients (browser extension, mobile, desktop, and the `bw` CLI), so setup is just standing up the server and pointing clients at it instead of bitwarden.com. The reason this matters for an AI/automation context specifically: OS-native password managers (Google Password Manager, Samsung Pass, Apple Keychain) have no CLI or API — an agent can only get a credential out of them via a human manually copying and pasting it, which briefly exposes it as plaintext outside the vault. `bw` avoids that relay step entirely.

## Questions to ask before starting

1. Docker available on the target host? (Docker is the simplest path; bare-metal install is possible but not the default here.)
2. What hostname will this be reached at — local-only, or public via a tunnel (e.g. Cloudflare Tunnel, see that blueprint if the user has it)?
3. Does the user want CLI/scripted access (for an agent or automation), or just the standard apps? If CLI access is wanted, plan for the `bw unlock` session-key step and ask how they want to handle re-unlocking after the session expires (manual, or a stored master password for full automation — flag the tradeoff either way).
4. Is the eventual hostname going to sit behind Cloudflare's proxy? If yes, warn now that the `bw` CLI can hit a compression (zstd) compatibility issue through Cloudflare and plan for a local reverse-proxy bypass for CLI traffic specifically (browser/mobile clients aren't affected).

## Steps to execute

1. `docker run -d --name vaultwarden -e DOMAIN=https://<hostname> -e SIGNUPS_ALLOWED=true -p <host-port>:80 -v <data-path>:/data --restart unless-stopped vaultwarden/server:latest`
   - Visit the hostname and create the first account before doing anything else.
2. Recreate the container (Docker env vars require this — stop/rm and rerun with updated flags, or use `docker-compose`) with `SIGNUPS_ALLOWED=false` and a long random `ADMIN_TOKEN` added — closes signups and unlocks `/admin` for user/org management in one pass.
3. Install `bw` CLI (npm or standalone binary). `bw config server https://<hostname>` (or the local bypass URL if the zstd issue applies), then `bw login`.
4. For automation: `bw unlock` returns a session key — export as `BW_SESSION`. `bw get password "<item name>"` retrieves a credential programmatically.
5. If Cloudflare-proxied and CLI calls fail with decompression/response errors: set up a local reverse proxy (nginx + self-signed/mkcert cert is enough) straight to the container's LAN port, add an `/etc/hosts` entry for a local-only hostname, and point `bw config server` at that instead. Do not change the public-facing hostname used by browser/mobile clients — this fix is CLI-only.

## Watch for

- `SIGNUPS_ALLOWED` must be turned off after the first account exists, or the instance is open to anyone who finds the URL.
- The admin token gates `/admin` only — losing it doesn't lose vault data, just panel access; reset by recreating the container with a new `ADMIN_TOKEN`.
- Full unattended automation requires storing the master password somewhere for periodic re-unlock — this is an unavoidable single root secret, so ask the user how they want to protect that one value (this is a real security decision, don't default it silently).
- Data lives in the mounted volume only — confirm the user has a backup plan for it before calling this done.

## Done when

- Vaultwarden container is running (`--restart unless-stopped`) and reachable at the chosen hostname.
- First account created, `SIGNUPS_ALLOWED=false` set.
- At least one client (browser, mobile, or `bw`) successfully logged in and can read/write an item.
- If CLI/agent access was requested: `bw get password` successfully returns a real stored credential from a script or shell, not just interactively.
