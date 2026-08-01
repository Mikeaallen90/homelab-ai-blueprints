# Vaultwarden — Free, Self-Hosted, Cross-Platform Password Manager

## Why This Exists

For those of us who use the command line a lot, or even AI in the command line, having access to our passwords is important — and securing them should be just as important. Vaultwarden is the free, self-hosted version of Bitwarden. The great thing about it is you can store your passwords from the web and your phone, all in one place. I don't know about other AI, but Claude can't access Google Passwords or Samsung Pass — so anything stored there would have to be copied and sent in plaintext for an AI to use it. Vaultwarden uses Bitwarden's official apps, so you can set it as your password manager on your phone, in your browser, and on the command line — and it can store more than just usernames and passwords.

## What It Does

Runs your own password manager server — same apps as Bitwarden (browser extension, mobile, desktop, CLI), same end-to-end encryption, but the vault lives on your hardware instead of Bitwarden's cloud. Syncs across every device you install a client on. Stores more than logins — secure notes, cards, identities all fit too.

The part that matters most for command-line or AI use: your phone's built-in password manager (Google Password Manager, Samsung Pass, Apple Keychain) has no CLI and no API a script or an AI agent can call. If an agent needs a credential, the only way to get it out of those is a human manually copying it and pasting it somewhere — which means the password briefly exists in plaintext outside the vault. Vaultwarden ships with `bw`, a real command-line client, so a script or an agent can pull a credential directly and programmatically, without a human relay step.

## What You'll Need

- Docker (or a bare-metal install, but Docker is simplest)
- A hostname to reach it from — either local-only, or public via [Blueprint #1](../01-Cloudflare-Tunnel/Guide.md) (Cloudflare Tunnel) if you want mobile/remote access
- The `bw` CLI installed wherever you want scriptable/AI access (npm, or standalone binary)

## How It Works

`vaultwarden/server` runs as a single Docker container, binding an internal port (80) to a host port you choose. All client apps — including `bw` — talk to it over that one HTTP(S) endpoint. The `DOMAIN` env var tells the server what URL to advertise back to clients (important for correct links/redirects). An admin panel is exposed at `/admin`, gated by a separate admin token, for user management and settings without needing a database console.

## Step-by-Step Setup

1. `docker run -d --name vaultwarden -e DOMAIN=https://your.hostname -e SIGNUPS_ALLOWED=true -p 8222:80 -v /path/to/data:/data --restart unless-stopped vaultwarden/server:latest`
2. Visit the hostname, create your one account.
3. Lock it down: recreate the container with `SIGNUPS_ALLOWED=false` and a long random `ADMIN_TOKEN` added (to unlock the `/admin` panel for user/org management) — Docker env vars require recreating the container to change, not just restarting, so stop/remove and rerun the same command with the updated flags (or use `docker-compose` so future changes are just a config edit + `docker compose up -d`).
4. Install a client — browser extension, mobile app, or `bw` CLI — point it at your server URL under Settings → self-hosted, instead of the default bitwarden.com.
5. For remote/mobile access, route the hostname through [Blueprint #1](../01-Cloudflare-Tunnel/Guide.md) (Cloudflare Tunnel) rather than opening a port on your router.
6. **For CLI/script/AI access**: `bw login` once interactively, then `bw unlock` returns a session key. Export it (`export BW_SESSION=...`), and from then on `bw get password "Item Name"` (or `bw get item`, `bw get note`, etc.) returns the credential directly — no manual copy-paste, no plaintext relay through you. The session key expires and needs periodic re-unlock; see Gotchas for the full-automation tradeoff.

## Adaptation Notes

- If you're not using Cloudflare Tunnel, any reverse proxy (nginx, Caddy, Traefik) with a valid TLS cert in front of the container works — Bitwarden clients require HTTPS except on localhost.
- Websockets (live push of vault changes to other logged-in devices) are on by default since Vaultwarden v1.29.0 — no env var needed. `ENABLE_WEBSOCKET=false` is the opt-out if you want to disable it. If you're running a reverse proxy in front, it must pass through the `Upgrade`/`Connection` headers or live push silently falls back to polling.
- Data lives entirely in the mounted volume — back that up, it's the whole vault (encrypted at rest, but still the only copy).
- Beyond logins: secure notes, credit cards, and identity records all live in the same vault and are reachable the same way from `bw`, so this doubles as general secure storage for an agent, not just a password store.

## Gotchas

- **Cloudflare zstd compatibility issue**: if your Vaultwarden domain is proxied through Cloudflare, the `bw` CLI (and some other Bitwarden clients) can choke on Cloudflare's zstd response compression — this is a documented class of problem, not just us. Fix: run a local reverse proxy (nginx with a self-signed/mkcert cert is enough) that proxies straight to the container's LAN port, and point `bw config server` at that local HTTPS URL instead of the public Cloudflare-proxied one. Browser/mobile clients using the public URL directly aren't affected — this is specifically a CLI/scripting workaround.
- **Full automation vs. security tradeoff**: `bw unlock` needs your master password, so a fully unattended agent needs that password stored somewhere to re-unlock after the session key expires — which means deciding how that one secret itself gets protected. There's no way around having one root secret somewhere; the goal is minimizing how many places it lives, not eliminating it.
- Losing the admin token means losing admin-panel access, not data — you can reset it by recreating the container with a new `ADMIN_TOKEN`.
- `SIGNUPS_ALLOWED=false` should go on right after your account exists, or the instance is open to anyone who finds the URL.
- **`bw list items` and `bw get item` always return full decrypted data — there's no metadata-only mode.** If you want to audit "what's in my vault" (names, folders, usernames) without touching passwords, you can't get that from the CLI alone; every call pulls full decrypted secrets. Pipe straight into `jq`/`grep` and discard immediately — never let the raw output touch a file, even a scratch one.
- **A bulk import from a browser or another password manager can massively over-duplicate** — Bitwarden/Vaultwarden's import has no de-dup step. One real case: an 866-item pile turned out to be the same export imported twice, four days apart. If you're migrating an existing password store in, check for and clear duplicates in the same session as the import, before the pile grows unmanageable.
- **A flat, unfoldered vault stops being usable well before you'd expect.** Once item counts get into the hundreds, "just search by name" breaks down. Set up folders proactively, and keep "infrastructure I administer" (the actual services you run) in its own folder, separate from personal/consumer and smart-home vendor accounts — otherwise the infra folder becomes a junk drawer and loses its usefulness as a quick reference.
- **Hash comparison is the safe way to check "does this plaintext copy match what's already in the vault"** without ever displaying either value: hash both (e.g. `sha256sum`) and compare only the hash output. Use this during any credential-migration/audit work to confirm duplicates, verify writes, and detect stale copies without a secret ever touching your terminal output. Caveat: matching hashes only prove two copies are identical, not that either still works — for anything where staleness is plausible (a token that might have rotated), test it live against the real service before trusting it.
- **The vault's own unlock password can't live inside the vault it unlocks** — it needs a backup plan outside it, and a bare text file is the weakest option available. If your OS has a built-in secret store (GNOME Keyring, Windows Credential Manager, macOS Keychain), use that instead of a markdown file for the human-readable backup copy — check what's actually installed first (the keyring daemon may already be running, but a CLI companion like `secret-tool` from `libsecret-tools` may need a separate install). Whatever tool reads that backup back later can have its own "dump everything" flag exactly like `bw` does (e.g. `secret-tool lookup` returns just the value, but `secret-tool search --all` prints it in plaintext by design) — check the exact command's output mode before running it, for every new tool you introduce, not just `bw`.
- **Overlapping/concurrent `bw` CLI invocations against the same local session can corrupt it** — e.g. a retried background call running a fresh unlock while a prior one is still mid-write can wipe the local CLI's config to a fully-logged-out state (server URL gone, not just a locked session). This is local CLI state corruption, not an account compromise. Recovery: reset the server config and re-authenticate; if a password-based re-login hangs on an interactive prompt, use a `--passwordfile`-style option instead of piping the password or using an env var, if the CLI supports it.
- `sed -i` against a network-mounted (SMB/GVFS) file prints a harmless `preserving permissions... Operation not supported` warning on every edit — expected noise for that mount type, not a real failure.
- If a bot token or similar credential turns up in an old note under a "legacy"/"retired" label, verify liveness against the platform's own lightweight check (e.g. Telegram's `getMe`) before assuming it's dead — a framework built *around* a credential can be fully retired while the credential itself is still live and reused elsewhere. Don't conflate "this doc section is stale" with "this value no longer works."
- If something is mistakenly deleted without a backup, check whether the underlying platform can just show it to you again (many services, e.g. BotFather for Telegram bots, let you re-view or regenerate a credential without a file-level restore) before assuming it's gone for good.
