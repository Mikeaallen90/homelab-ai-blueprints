# Blueprint #6 — Audiobook Pipeline for Family Sharing

## 0. Why This Exists

Lots of people love audiobooks, and apps like Audible preach that a book you buy is yours for life. If it's yours, take control of it. I can't say what happens if Audible itself ever goes away, and I won't try to overstate the power of owning your own stuff — but I'm a big supporter of it.

Yes, you can share your books by adding people to your Amazon account — but that means adding others to your actual Amazon account. The OpenAudible + Audiobookshelf combo does the same thing safely, and a lot more: everyone gets their own account and their own listening progress, without anyone touching your Amazon login.

It's also a great way to have friends and family listen to each other's books together, or even start an audiobook club.

## 1. What It Does

OpenAudible logs into your Audible account, tracks your whole purchased library, and downloads + converts each title from Audible's DRM'd format into a standard, DRM-free file — your choice of MP3 or M4B — along with cover art and metadata. It can have multiple Audible accounts attached at once, so a household with more than one Audible subscriber can pull everyone's books into one place.

A symlink step then publishes only the finished audio files + art into a separate library folder — Audiobookshelf never touches your Audible credentials or OpenAudible's own working folder.

Audiobookshelf serves that folder as a real audiobook app: individual accounts per listener, per-person progress tracking that syncs across devices, downloads for offline listening, series/metadata handling, and mobile + web apps.

## 2. What You'll Need

- Active Audible subscription (this is the "buy" side — the pipeline doesn't get you free books)
- OpenAudible (desktop app, `.deb` available) — **not free**: about $22/year, or about $80 as a one-time lifetime license
- Audiobookshelf (free, self-hosted, Docker) — the family-facing server
- A machine to run both, reachable to family (LAN-only works if everyone's home; add [Blueprint #1](../01-Cloudflare-Tunnel/Guide.md) Cloudflare Tunnel for access away from home)
- Meaningful storage — audiobooks run 100MB–1GB+ each
- A little time with Audiobookshelf's own docs — its library folder structure is specific to how it expects things organized, worth understanding before you're deep into automation

## 3. How It Works

1. OpenAudible authenticates to Audible (one or more accounts), syncs your library metadata
2. It downloads + converts each title to MP3 or M4B, with cover art and metadata, into its own private folder
3. A symlink script scans for new audio files not yet linked, and links them into a separate folder that's Audiobookshelf's actual library root — real files exist once, only pointers get duplicated
4. Audiobookshelf scans that folder and exposes it as a browsable/streamable library with per-user logins

## 4. Step-by-Step Setup

### Quick Version (just want it working)

1. Install OpenAudible, pay for it (~$22/yr or ~$80 lifetime), log into your Audible account, and turn on auto-download + auto-sync in its settings
2. Install Audiobookshelf via Docker, point it at a library folder
3. Whenever you buy something new, run (or schedule) a symlink script that links newly-downloaded files from OpenAudible's folder into Audiobookshelf's library folder
4. Create Audiobookshelf accounts for family/friends — done

### Full Version (more control, matches how this was actually built)

1. Confirm/get an Audible subscription (one per person whose books you want included, if more than one)
2. Buy and install OpenAudible, log in each Audible account you want attached
3. In OpenAudible's settings, turn on auto-download and auto-sync so new purchases get pulled in without you triggering it manually each time
4. Read through Audiobookshelf's docs on library folder structure before setting up its library folder — getting this right up front saves reorganizing later
5. Create a separate "public" folder that will be Audiobookshelf's library root — deliberately not the same folder OpenAudible uses, to keep your Audible auth/raw library isolated from what family can browse
6. **Test one real symlink now, before building automation around it**: `ln -s` one finished file from OpenAudible's folder into the public folder. If the library lives on network-mounted storage (CIFS/GVFS) and this fails with "Operation not supported," you'll need to SSH directly into the storage device and create the symlink there using its own local filesystem path instead (see Gotchas) — better to find that out now than after the automation script is built around an assumption that doesn't hold.
7. Build a small automation script that: checks for new purchases/updates → opens OpenAudible if it isn't already running (relying on the auto-download/sync you set up in step 3) → waits, since download+convert time varies a lot depending on how many books are queued → creates symlinks for any newly finished files into the public folder
8. The one manual step that's left: creating the actual series/organization folders inside Audiobookshelf's library location, and moving the new symlinks into them
9. Install Audiobookshelf via Docker, mount the public folder + config/metadata volumes
10. Create individual Audiobookshelf accounts for each family member or friend
11. Optional: expose via Cloudflare Tunnel ([Blueprint #1](../01-Cloudflare-Tunnel/Guide.md)) for access away from home
12. Schedule step 7's script (cron, systemd timer, or an agent skill) so new purchases keep flowing in automatically

## 5. Adaptation Notes

- The underlying shape — download in a protected format → convert to standard format → symlink into a serving app — applies even if someone's not on Audible
- Audiobookshelf can be swapped for another self-hosted audiobook server; the key pattern is keeping "owns the account" separate from "serves the listeners"
- Multiple Audible accounts can feed one OpenAudible instance — useful for a household where more than one person has their own subscription and everyone wants a combined library
- This doesn't have to stay inside one family — an audiobook club or friend group can run the same setup to share a combined library
- Symlinking is what avoids doubling storage, but it doesn't work over every filesystem — flagging as a gotcha below since it's a real one hit during setup

## 6. Gotchas

- Stripping DRM from books you bought sits in a legal gray area in some places, even though you own the content — worth knowing your local law if that matters to you. This whole approach rests on the idea that "yours for life" should actually mean yours.
- Real symlinks failed over a CIFS/GVFS network mount ("Operation not supported") — had to SSH directly into the storage device and run `ln -s` there instead. If the library lives on network storage, plan for this.
- OpenAudible's own status file is a snapshot, not live — a status-check built on it can report stale info mid-download.
- No automatic series/metadata sorting — new titles land flat, and organizing into folders is a manual step by design.
- Audiobookshelf accounts are separate from Audible — nothing carries over, everyone needs a login created for them.
