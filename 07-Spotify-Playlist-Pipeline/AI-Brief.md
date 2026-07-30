# Blueprint #7 — Spotify Playlist → Self-Hosted Library + Plex Playlist (AI Brief)

## Purpose

Build a pipeline that downloads a Spotify playlist's tracks into a deduplicated shared music library, links them into a named playlist folder without duplicating storage, verifies the files, and registers a real playlist in the user's media server. See `Guide.md` for the full human-readable version, including a link to spotDL's own docs — this file is for an AI assistant executing the setup on the user's behalf.

## Questions to Ask Before Starting

1. Where does the user's music library live (local disk, NAS, other network storage), and is it mounted in a way that supports creating real symlinks? Test this early — if it fails, the fix is connecting to the storage device directly (e.g. SSH) rather than through the mounted path.
2. What media server are they using — Plex, or something else? Confirm how that server's playlist API actually works (don't assume an `.m3u` import endpoint works even if it returns success — verify with one small test playlist first).
3. Is there already a single shared "all songs" folder, or does one need to be created? This is the piece that prevents duplicate downloads across playlists — don't skip it.
4. Is the playlist to download actually a playlist, or Liked Songs? If Liked Songs, tell the user to duplicate it into a real playlist first and get that URL instead.
5. Does the user want full automation (a webhook that takes just a playlist URL and runs everything), or is manual/on-demand running fine for now?
6. Do they have (or want) a malware scanner and ffprobe/ffmpeg available for the verification steps, or should those be skipped/simplified?

## What "Done" Looks Like

- A shared "all songs" library folder exists, and running the pipeline against a real Spotify playlist URL downloads only tracks not already present in it.
- A playlist folder of symlinks (not copies) exists, pointing back into the shared folder, matching the playlist's real track list.
- The malware scan and audio-only verification both run and report clean (or report exactly what's wrong, not silently pass a bad file) — unless the user answered Q6 to skip/simplify them, in which case confirm they explicitly understand downloaded files are being accepted unverified.
- The playlist shows up as a real, named, browsable playlist in the user's media server, with a track count matching the downloaded/matched count.
- If full automation was requested: sending a playlist URL to the webhook alone (no other manual steps) produces a finished, verified, registered playlist.

## Key Points to Carry Into the Build

- One shared library folder is the whole mechanism that prevents duplicate downloads — every playlist is just symlinks into it, never its own copy of any file. Don't let a playlist folder become anywhere audio actually gets downloaded to.
- Get the authoritative track list directly from Spotify before downloading, and use it afterward to determine exactly which downloaded files belong to this playlist — don't try to infer this from download log output, which can be misleading (e.g., an error line adjacent to an unrelated URL).
- Liked Songs isn't a normal playlist for most external tools — always confirm the user is passing a real playlist URL, and if it's Liked Songs, have them duplicate it into a real playlist first.
- Test symlink creation on the actual storage early. If it fails over a network mount, don't keep retrying the same way — switch to creating the symlink by connecting to the storage device directly.
- Don't pipe a long-running download process through anything that could close the pipe early (like `head`) — this can silently kill the download while it still reports a clean exit. Redirect to a log file and monitor separately instead.
- Do exactly one retry pass on failed tracks after the main download, then stop and report the remainder as genuine failures — don't retry indefinitely.
- Before trusting a media server's built-in playlist-import feature, verify it actually works with one small test playlist — some versions return success without actually doing anything.
- Scan the whole shared library on each run, not just newly added files — catches anything that arrived through a different route since the last scan.
- If the user wants full automation, the whole flow (get track list → download → match → symlink → verify → register playlist → final library scan) should be wrapped so that a webhook call with just a playlist URL (and optionally a playlist name) triggers all of it end to end.
- Point the user at spotDL's own documentation (https://spotdl.readthedocs.io/en/latest/) for current flags/output-format options rather than hardcoding assumptions from this brief — those change between tool versions.
