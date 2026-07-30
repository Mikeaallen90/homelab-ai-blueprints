# Cloudflare Tunnel — AI Brief

Read this, then ask the user the questions below before doing anything. Goal: get their self-hosted service(s) reachable at a public hostname, with no inbound port forwarding on their router.

## Context

Cloudflare Tunnel runs `cloudflared` as a background service that opens outbound-only connections to Cloudflare's edge. Public hostnames (subdomains of a domain the user controls on Cloudflare) route through that tunnel to local `ip:port` services. No inbound firewall rule is ever needed. Requires the user already own a domain and have it added to Cloudflare (nameservers pointed at Cloudflare) — if they don't, that's step zero, not optional.

## Questions to ask before starting

1. Do you already have a domain on Cloudflare? If not, help them pick a registrar first (Cloudflare Registrar for at-cost pricing with no renewal jump, Porkbun/Namecheap for cheapest entry — warn that promo pricing isn't the renewal price, Hostinger if they want domain+hosting+email together) and get nameservers pointed at Cloudflare before continuing.
2. Which local service(s) do you want to expose, and on what port(s)?
3. Does any backend use a self-signed TLS cert (e.g. a NAS admin panel)? If yes, that ingress entry needs `originRequest: noTLSVerify: true`.
4. What OS/package manager is the host running (for the `cloudflared` install step)?
5. Dashboard-managed or config-file-managed? Default to config-file-managed (Option B) if they're managing more than one or two services, or want the setup to be scriptable/auditable/backed up in one file. Default to dashboard-managed (Option A) if they want the fastest path to one service and aren't comfortable editing YAML.

## Steps to execute

**If Option A (dashboard-managed):**
1. Have them create the tunnel in Cloudflare dashboard → Networking → Tunnels.
2. Run the OS-specific install-and-connect command it generates.
3. Confirm the connector shows active in the dashboard.
4. Add a "Public Hostname" entry per service, pointing to its local `ip:port`.

**If Option B (config-file-managed):**
1. Install `cloudflared` for the host's OS/package manager (the answer to Q4).
2. `cloudflared tunnel login`
3. `cloudflared tunnel create <name>`
4. Write `config.yml`:
   ```yaml
   tunnel: <uuid>
   credentials-file: /path/to/<uuid>.json
   ingress:
     - hostname: service1.yourdomain.com
       service: http://127.0.0.1:PORT
     - service: http_status:404
   ```
5. `cloudflared tunnel route dns <tunnel> <hostname>` for each hostname.
6. `cloudflared service install`, then enable/start it as a system service (systemd: `systemctl enable --now cloudflared`).

## Watch for

- The `ingress` list MUST end with `service: http_status:404` or the config is invalid.
- Self-signed backend certs need `noTLSVerify: true`, not a switch to plain HTTP.
- Don't create a new tunnel per service — one tunnel, multiple `ingress` entries (Option B) or multiple Public Hostnames (Option A).
- Losing the credentials JSON (Option B) with no backup means recreating the tunnel from scratch.

## Done when

- `cloudflared` is running as a persistent background service (survives reboot).
- Each requested hostname resolves and successfully reaches its local service.
- Tested from a network outside the user's LAN (e.g. cellular data), not just locally.
