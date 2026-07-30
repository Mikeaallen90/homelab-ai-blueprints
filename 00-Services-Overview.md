# Services Overview

**What this is:** A single reference list of every free/self-hostable service, tool, or app referenced across the Blueprints collection — Cloudflare Tunnel, Vaultwarden, n8n, and so on. Each gets a short "what it is" and "why a project here needs it" blurb, plus a verified download/homepage link.

**How it's useful:** If you're reading a blueprint and hit an unfamiliar name, you don't have to stop and go research it or hunt through every other blueprint that happens to mention it — look it up here instead. This file is *not* setup instructions; those live in whichever blueprint actually uses the service, so nothing is duplicated. It grows incrementally — a new entry gets added whenever a new blueprint introduces a service that isn't listed yet.

---

## Networking & Domains

**Cloudflare Tunnel (cloudflared)** — [Downloads/install docs](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/)
A client that opens outbound-only tunnels from a host to Cloudflare's edge, exposing local services on public subdomains without port-forwarding. Used across most blueprints ([#1](01-Cloudflare-Tunnel/Guide.md), [#2](02-Vaultwarden-Password-Manager/Guide.md), [#3](03-Agent-Creation/Guide.md), [#6](06-Audiobook-Pipeline/Guide.md), [#8](08-Email-Archiving/Guide.md)) so self-hosted services can be reached remotely/on mobile without opening router ports.

**Cloudflare Registrar** — [domains.cloudflare.com](https://domains.cloudflare.com/)
Cloudflare's at-cost domain registration. Convenient since it lives in the same account as the tunnel ([#1](01-Cloudflare-Tunnel/Guide.md)).

**Hostinger** — [hostinger.com/domains](https://www.hostinger.com/domains)
Commercial domain/hosting/email provider. An alternative domain source that bundles hosting and email together ([#1](01-Cloudflare-Tunnel/Guide.md)).

**Porkbun** — [porkbun.com](https://porkbun.com/)
Budget domain registrar. A cheap first-year option for getting a domain ([#1](01-Cloudflare-Tunnel/Guide.md)).

**Namecheap** — [namecheap.com](https://www.namecheap.com/)
Budget domain registrar, same role as Porkbun ([#1](01-Cloudflare-Tunnel/Guide.md)).

**DuckDNS** — [duckdns.org](https://www.duckdns.org/)
Free dynamic DNS service. Lets you test the tunnel pattern with a free subdomain before buying a real domain ([#1](01-Cloudflare-Tunnel/Guide.md)).

**No-IP** — [noip.com/free](https://www.noip.com/free)
Free dynamic DNS service, same role as DuckDNS ([#1](01-Cloudflare-Tunnel/Guide.md)).

## Secrets & Credentials

**Vaultwarden** — [GitHub repo](https://github.com/dani-garcia/vaultwarden)
A free, self-hosted, Bitwarden-compatible password/secrets server (with a `bw` CLI). Gives scripts and AI agents programmatic, non-plaintext credential access instead of manual copy/paste ([#2](02-Vaultwarden-Password-Manager/Guide.md), [#3](03-Agent-Creation/Guide.md), [#8](08-Email-Archiving/Guide.md)).

## Infrastructure

**Docker** — [Docker Desktop](https://www.docker.com/products/docker-desktop/) · [Engine-only install docs](https://docs.docker.com/engine/install/) (for servers)
A container runtime. The simplest way to run services like Vaultwarden and Audiobookshelf as isolated, restart-safe containers ([#2](02-Vaultwarden-Password-Manager/Guide.md), [#6](06-Audiobook-Pipeline/Guide.md), [#9](09-SMS-Archive-And-Search/Guide.md)). Desktop is free for personal/small-business use; Pro/Team/Business are paid tiers.

**SQLite** — [sqlite.org/download](https://sqlite.org/download.html)
A free embedded database engine, no server process required. Stores imported messages or archived emails so an agent can query/search them quickly instead of grepping raw files ([#9](09-SMS-Archive-And-Search/Guide.md), and email archiving in [#8](08-Email-Archiving/Guide.md)).

## Agents & Automation

**Claude Code** — [code.claude.com/docs](https://code.claude.com/docs/en/overview)
Anthropic's CLI-based AI agent tool. The engine that gives each directory-scoped agent its identity, memory, and remote-control/session features ([#3](03-Agent-Creation/Guide.md), [#4](04-Multi-Agent-Interaction/Guide.md)).

**Telegram** — [App](https://telegram.org/apps) · [Bot API docs](https://core.telegram.org/bots/api)
A messaging platform with a bot API. The chat channel used for reaching agents remotely and for watchdog/self-report alerts ([#3](03-Agent-Creation/Guide.md), [#5](05-Scheduled-Self-Reporting/Guide.md)).

**n8n** — [n8n.io](https://n8n.io/)
A free, self-hostable workflow-automation tool with a visual editor and credential store. Wires Telegram triggers, scheduled health/update checks, and webhook-driven pipelines like Spotify downloads and email archiving ([#3](03-Agent-Creation/Guide.md), [#5](05-Scheduled-Self-Reporting/Guide.md), [#7](07-Spotify-Playlist-Pipeline/Guide.md), [#8](08-Email-Archiving/Guide.md)). Note: licensed "fair-code," not fully open source.

## Mobile Widget & Webhook Triggers

**Tasker** — [Play Store](https://play.google.com/store/apps/details?id=net.dinglisch.android.taskerm)
The flagship paid Android automation app — home-screen widgets, HTTP requests, and conditional logic to trigger things like n8n webhooks from your phone.

**MacroDroid** — [Play Store](https://play.google.com/store/apps/details?id=com.arlosoft.macrodroid)
A free alternative to Tasker (capped at 5 macros free; one-time payment unlocks unlimited, no subscription). Same widget/HTTP-trigger role.

**HTTP Shortcuts** — [GitHub](https://github.com/Waboodoo/HTTP-Shortcuts)
A free, open-source, ad-free app purpose-built for firing HTTP requests from a home-screen widget. Pairs well alongside Tasker or MacroDroid when all you need is "tap to hit a webhook."

## Audiobook Pipeline

**Audible** — [audible.com](https://www.audible.com/)
Amazon's audiobook subscription/purchase service. The source library that OpenAudible pulls owned titles from ([#6](06-Audiobook-Pipeline/Guide.md)).

**OpenAudible** — [openaudible.org](https://openaudible.org/)
A paid desktop app (free trial available) that logs into Audible and converts purchased audiobooks to DRM-free MP3/M4B. The piece that actually liberates bought audiobooks from Audible's app/DRM for personal family serving ([#6](06-Audiobook-Pipeline/Guide.md)).

**Audiobookshelf** — [audiobookshelf.org](https://www.audiobookshelf.org/)
A free, self-hosted audiobook/podcast server with per-user accounts and progress sync. Serves the converted library to family/friends without sharing Audible credentials ([#6](06-Audiobook-Pipeline/Guide.md)).

## Music Pipeline

**spotDL** — [GitHub repo](https://github.com/spotDL/spotify-downloader)
A free, open-source tool that reads public Spotify playlist metadata and downloads matching audio elsewhere. Turns a Spotify playlist into local files for the shared library ([#7](07-Spotify-Playlist-Pipeline/Guide.md)).

**Plex** — [plex.tv downloads](https://www.plex.tv/media-server-downloads/)
A free-tier media server. Hosts and serves the downloaded/symlinked music as a real, browsable playlist ([#7](07-Spotify-Playlist-Pipeline/Guide.md)).

**ClamAV** — [clamav.net/downloads](https://www.clamav.net/downloads)
A free antivirus/malware scanner. Verifies downloaded audio files aren't malicious before they're added to the library ([#7](07-Spotify-Playlist-Pipeline/Guide.md)).

**ffmpeg (includes ffprobe)** — [ffmpeg.org/download](https://ffmpeg.org/download.html)
A free media-inspection/conversion toolkit. Confirms downloaded files are genuinely audio, not mislabeled video ([#7](07-Spotify-Playlist-Pipeline/Guide.md)).

## Email Archiving

**mbsync / isync** — [isync.sourceforge.io](https://isync.sourceforge.io/)
A free, open-source IMAP mail-sync tool. Pulls the full mailbox down to local staging before archiving and deleting from the provider ([#8](08-Email-Archiving/Guide.md)). Canonical home is SourceForge, not GitHub.

## SMS Archiving

**SMS Backup & Restore** — [Play Store](https://play.google.com/store/apps/details?id=com.riteshsahu.SMSBackupRestore)
A free Android app that exports SMS/MMS history to a cloud drive. The source feed for the personal, queryable text-message archive ([#9](09-SMS-Archive-And-Search/Guide.md)).

**Google Drive** — [drive.google.com](https://www.google.com/drive/)
Cloud storage. Bridges the phone's backup exports to a location the AI agent's machine can actually read ([#9](09-SMS-Archive-And-Search/Guide.md)).

**Synology Cloud Sync** — [synology.com package page](https://www.synology.com/en-global/dsm/packages/CloudSync)
NAS-side sync tooling that pulls the Google Drive backup down to local/NAS storage ([#9](09-SMS-Archive-And-Search/Guide.md)).
