# Blueprint #6 — Audiobook Pipeline for Family Sharing (AI Brief)

## Purpose

Set up a pipeline that downloads a user's purchased Audible audiobooks into DRM-free files via OpenAudible, then serves them to family/friends as their own app (individual accounts, progress tracking) via Audiobookshelf — without anyone needing the user's Audible/Amazon login. See `Guide.md` for the full human-readable version, including a Quick Version and a Full Version of setup — this file is for an AI assistant executing the setup on the user's behalf.

## Questions to Ask Before Starting

1. Does the user already have an Audible subscription, and if more than one account should feed the library, how many and whose?
2. Has OpenAudible been purchased/installed yet? (It's a paid app — confirm the user is aware, ~$22/yr or ~$80 lifetime, before proceeding.)
3. Preferred output format — MP3 or M4B?
4. What machine will run OpenAudible + Audiobookshelf, and is it reachable to family only on the LAN, or does it need remote access ([Blueprint #1](../01-Cloudflare-Tunnel/AI-Brief.md), Cloudflare Tunnel)?
5. Where should Audiobookshelf's library folder live, and has the user looked at Audiobookshelf's own docs on its expected folder structure? (Worth doing before building automation around it.)
6. Does the user want the Quick Version (minimal, manual re-run when new books arrive) or the Full Version (scheduled automation script) from the Guide?
7. Is the storage the library lives on a network mount (CIFS/GVFS/SMB)? If so, flag the known symlink limitation early (see Gotchas) — real symlinks may need to be created by SSHing into the storage device directly rather than through the mounted path.
8. Who should get Audiobookshelf accounts (names/relationships), so those can be created at the end?

## What "Done" Looks Like

- OpenAudible is installed, logged into the intended Audible account(s), and set to auto-download + auto-sync.
- A separate library folder exists that Audiobookshelf reads from, kept distinct from OpenAudible's own working folder.
- New audiobooks download and convert in OpenAudible, then get symlinked (not copied) into the Audiobookshelf library folder — verify with at least one real title end to end.
- Audiobookshelf is running via Docker, scanning that folder, and has a working account created for at least one family member besides the owner.
- If the Full Version was chosen: a scheduled job (cron/systemd timer/agent skill) runs the check-download-wait-symlink sequence without manual triggering, and the user understands the one remaining manual step (creating series folders and moving new symlinks in).
- If remote access was requested: the Audiobookshelf URL is reachable through the Cloudflare Tunnel and works from outside the LAN.

## Key Points to Carry Into the Build

- Keep OpenAudible's own folder and Audiobookshelf's library folder separate — this isolates Audible credentials/raw library data from what's exposed to other users. Don't point Audiobookshelf directly at OpenAudible's working folder even if it seems simpler.
- Symlink, don't copy — real files exist once, only pointers get duplicated. If the library lives on network-mounted storage (CIFS/GVFS), test whether `ln -s` actually works over that mount before building automation around it; if it fails with "Operation not supported," the fix is SSHing directly into the storage device and creating the symlink there using its own local filesystem path, not the mounted path.
- OpenAudible must have auto-download + auto-sync turned on in its own settings for any automation script to be useful — otherwise "open the app" doesn't actually fetch anything new.
- Any automation script needs a wait/delay step after triggering OpenAudible — download+convert time scales with how many books are queued, and checking for finished files too early will miss them.
- OpenAudible's status/library file is a point-in-time snapshot, not live — don't build a status check that assumes it reflects an in-progress download accurately.
- Folder organization (series, etc.) inside Audiobookshelf's library is intentionally left as a manual step — don't try to auto-sort; just get new files symlinked in flat and let the user (or a later manual pass) organize.
- Flag the DRM-stripping legal gray area plainly once, briefly — this pipeline is built on the premise that content the user purchased is genuinely theirs to control, not a workaround to avoid mentioning.
- If asked to support multiple Audible accounts, confirm OpenAudible is configured for each before assuming a combined library will work as expected.
