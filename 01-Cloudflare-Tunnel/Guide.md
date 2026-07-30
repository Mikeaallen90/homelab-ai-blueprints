# Cloudflare Tunnel — Expose Self-Hosted Services Privately

## Why This Exists

To get the most out of a lot of these services, having your own domain is a must. I'd personally say get yourself a website with a custom domain so you can use its subdomains — but only if you have the money for the yearly costs. There are free options as well. You'll see a lot of my projects come back to having a domain, so it's just as important to have as a storage location. Here are some steps to get you started.

> Most other blueprints in this series assume you already have a domain on Cloudflare — this one covers getting that in place first.

## What It Does

Runs a small background service (`cloudflared`) that opens outbound-only connections from your machine to Cloudflare's network. You then map subdomains (e.g. `plex.yourdomain.com`) to local ports, and Cloudflare routes public traffic to them through that tunnel. No inbound port forwarding, your home/server IP is never exposed, and you get free TLS on every subdomain.

## What You'll Need

- **A domain name — the actual prerequisite.** Cloudflare Tunnel is just what you do once you have one; most of the other blueprints in this series assume this one already exists.
  - **Cloudflare Registrar** — sells at cost, no markup (they make money on the tunnel/CDN side instead), roughly $10–11/yr for `.com`, same price at renewal — no surprise price jump later. Simplest if the domain and the tunnel live in one account anyway.
  - **Hostinger** — where mine came from, alongside my website. Good if you want domain + hosting + email in one dashboard rather than juggling accounts.
  - **Porkbun / Namecheap** — often the cheapest entry price, especially on non-`.com` TLDs (sometimes under $2 the first year). These low prices are usually first-year-only promos — check the renewal price before committing, not just the signup price.
  - **Free alternative to test the pattern first**: a dynamic DNS service (DuckDNS, No-IP) gives you a free subdomain if you're not ready to pay for a domain yet. Fine for trying the tunnel setup itself, but you lose the "your own subdomains for everything" benefit that makes owning a domain worth it long-term.
- `cloudflared` installed on the host (apt/deb, brew, or binary — no Docker required, though it can run containerized)
- One config file + one credentials file per tunnel (Option B only, see below)

## How It Works

`cloudflared` authenticates once, creates a tunnel with a UUID, and gets a credentials JSON. A single `config.yml` maps `tunnel: <uuid>` + `credentials-file:` to an `ingress:` list of hostname → local-service rules, ending in a catch-all `http_status:404`. The service runs as a systemd unit so it survives reboots and reconnects automatically; it maintains several persistent QUIC connections to different Cloudflare edge locations for redundancy.

## Step-by-Step Setup

Cloudflare offers two ways to set this up. Both are fully supported — pick based on what fits your situation.

### Option A — Dashboard-managed (remotely-managed tunnel)

1. Cloudflare dashboard → Networking → Tunnels → Create a tunnel → name it.
2. Choose your OS, copy the install-and-connect command it gives you, run it on your server.
3. Wait for the connection to show as active in the dashboard.
4. Add each hostname as a "Public Hostname" in the dashboard, pointing to a local `ip:port`.

- Fastest to get one service running — no local config file to write.
- Config lives in the cloud, so it survives a full OS reinstall on the origin machine.
- No terminal editing once it's connected — good if you're not comfortable hand-editing YAML.
- Every hostname change is a dashboard click, not something you can script or version-control.
- Harder to review "everything this tunnel exposes" at a glance if you're managing several services.

### Option B — Config-file managed (locally-managed tunnel) — what we actually run

1. Install `cloudflared` for your OS/package manager (apt/deb, brew, or binary).
2. `cloudflared tunnel login` — authorizes against your Cloudflare account, drops a cert.
3. `cloudflared tunnel create <name>` — generates a UUID + credentials JSON in `~/.cloudflared/`.
4. Write `config.yml` with `tunnel:`, `credentials-file:`, and an `ingress:` list — one entry per service, ending in the required `service: http_status:404` catch-all.
5. `cloudflared tunnel route dns <tunnel> <hostname>` — once per hostname, creates the CNAME automatically.
6. `cloudflared service install`, then `systemctl enable --now cloudflared`.

- One file shows every service you expose — easy to audit, back up, or put in git.
- Scales cleanly to many services on one host without repeated dashboard clicks.
- Scriptable/repeatable — you can stand up the same tunnel on a new box from the file.
- Requires comfort editing YAML and using the terminal.
- If you lose the credentials JSON without a backup, you have to recreate the tunnel.

Both routes end the same way: `cloudflared` running as a background service, hostnames resolving through Cloudflare's edge to your local ports, nothing inbound on your router.

## Adaptation Notes

- If a backend service uses a self-signed cert (e.g. a NAS admin UI over HTTPS), add `originRequest: noTLSVerify: true` under that ingress entry instead of switching it to plain HTTP.
- Multiple services on one host = multiple `ingress` entries in one tunnel/config, not one tunnel per service.
- Works identically for a Raspberry Pi, a VPS, or a home server — the only requirement is that `cloudflared` can reach the internet outbound.

## Gotchas

- The ingress list is order-sensitive and **must** end with the `http_status:404` catch-all or the config is invalid.
- Cloudflare's free plan does rate-limit and cache aggressively for some content types — fine for dashboards/APIs, worth checking for anything serving large media.
- Delete/replace old tunnels you're not using — an abandoned tunnel with valid credentials sitting in your Cloudflare account is a stale credential, not a security hole by itself, but it's clutter worth pruning.
- Don't reuse one tunnel's credentials file across multiple machines unless you mean to — each tunnel is meant to represent one origin.
- Always check renewal pricing before picking a registrar — first-year promo pricing is not what you'll pay long-term.
