# Blueprint #7 — Spotify Playlist → Self-Hosted Library + Plex Playlist

## 0. Why This Exists

I'll keep this simple: with this, you can download your Spotify playlists. I suggest copying your Liked Songs into a separate playlist first before running this on it — Liked Songs doesn't behave like a normal playlist for a lot of external tools.

I use Plex to listen to music and store it on my NAS — you may be set up differently, but the shape of the setup will be similar, and your AI can help adapt it.

I suggest having one location for all your songs, because that's what stops duplicate downloads of the same song across different playlists. Combined with a script, you can keep your playlists intact. And combined with parameters and a webhook, you can even automate the whole process down to just sending a playlist URL.

I'd suggest reading the docs for the download tool for a clearer understanding — linked below.

## 1. What It Does

Reads a Spotify playlist's track list (artist/title per song) as the source of truth for what should end up downloaded. Downloads matching audio for every track into one shared library folder — if a song's already there from a different playlist, it's skipped, not re-downloaded. Builds a playlist folder of symlinks pointing back at the shared library, so the same song can belong to multiple playlists without ever being stored twice. Verifies the results (malware scan, confirms files are audio-only not disguised video) before registering a real, browsable Plex Playlist.

## 2. What You'll Need

- spotDL (free, open source) — reads Spotify's public playlist metadata and finds matching audio elsewhere to download; no Spotify Premium or API key needed for public playlists. Docs: https://spotdl.readthedocs.io/en/latest/
- A Plex server (free tier is enough) — or another media server if you're not on Plex
- Shared storage with a single-copy-per-song library folder, big enough for a real music collection
- ClamAV or similar (free) for the malware-scan verification step
- ffprobe (comes with ffmpeg, free) for the audio-only check
- Optional: a webhook/automation tool (e.g., n8n, free) if you want to trigger the whole pipeline just by sending a playlist URL

## 3. How It Works

1. Pull the playlist's real track list from Spotify (this is the ground truth — not scraped from download logs, which can be misleading)
2. Download every track into one shared "all songs" folder — the downloader itself skips a track if a matching filename already exists, which is what prevents re-downloading a song already pulled in for a different playlist
3. Compare the track list against what actually landed in the folder to know exactly which files belong to this playlist, and which genuinely failed
4. Build a playlist-named folder of symlinks pointing back at the real files in the shared folder — no duplicate storage, ever
5. Malware-scan the whole shared library (not just new files) and confirm every new file is genuinely audio, not a mislabeled video
6. Register it in Plex as a real playlist so it's browsable/streamable like anything else in your library

## 4. Step-by-Step Setup

1. Install spotDL and a media server (Plex or equivalent)
2. Set up one shared "all songs" folder — this is where every track from every playlist actually lives
3. Get the Spotify playlist's public share URL. If it's your Liked Songs, copy/duplicate it into a real playlist first and use that URL instead
4. Pull the authoritative track list from that URL
5. Download directly into the shared folder (never into a playlist-named folder — that's what causes duplicate copies)
6. Compare downloaded files against the track list to determine matched vs. failed tracks; do one retry pass for failures, then stop — don't retry indefinitely
7. Create a playlist folder of symlinks pointing back into the shared folder for just this playlist's tracks
8. Run a malware scan across the whole shared library, and confirm each new file is audio-only
9. Register the playlist with your media server so it shows up as a real, named playlist
10. Do a final library scan/sync so everything (new files, new playlist, folder changes) shows up correctly
11. Optional: wire steps 3–10 behind a webhook (e.g., n8n) so running the whole pipeline is as simple as sending a playlist URL, with the playlist name as an optional second parameter — no manual step-by-step needed after that's set up

## 5. Adaptation Notes

- If your storage is on a network mount, creating symlinks may fail the same way it did here — the reliable fix is creating them by connecting to the storage device directly (e.g., SSH) rather than through the mounted path
- If you're not on Plex, most self-hosted media servers support building a playlist from a folder of files or an import list — the "playlist = folder of symlinks into a single shared library" shape carries over regardless of server
- If your media server double-indexes the shared "all songs" folder in addition to each playlist folder, most servers support an ignore-file to exclude it from the main scan (Plex does)
- Read spotDL's own docs before adapting the download step — flags/output formats change between versions, and the docs are the current source of truth, not this blueprint

## 6. Gotchas

- These are downloaded from wherever spotDL finds a matching audio source (typically YouTube), not pulled from Spotify itself — Spotify's own streams stay licensed to Spotify. This is closer to building a personal backup/offline copy of music you already listen to than "owning" the files outright — worth knowing where you land on that before building a big library this way
- Spotify's Liked Songs list doesn't behave like a normal playlist for a lot of external tools — copy it into a real playlist first rather than pointing the tool at Liked Songs directly
- A media server's own playlist-import feature (like an `.m3u` upload endpoint) may be broken or dead on some versions even though it returns success — worth testing with one small playlist before assuming it works, rather than trusting it silently
- If storage is network-mounted, real symlinks may just fail outright with an unhelpful error — plan for the direct-connection workaround up front, not after hitting it
- Don't pipe a long-running download through something that closes the pipe early (like `head`) — it can kill the download silently while still reporting a clean exit
- Some downloads may need an extra runtime dependency depending on the audio source found — if a batch fails with a missing-dependency-shaped error, installing it and re-running just the failed subset is usually enough; the tool should skip files already downloaded
