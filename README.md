# iptv-epg

Self-updating unified XMLTV EPG for the IPTV playlist. Merges per-region guides from [epgshare01.online](https://epgshare01.online/epgshare01/) with the provider's own EPG, backfills the user's M3U channel ids, and adds a "No EPG" dummy programme for any remaining unmatched channel — so every entry in the M3U has data in the player grid.

## What to paste into your player

**EPG URL** (recommended — 12 MB gzipped, full programme descriptions):

```
https://al7omed.github.io/iptv-epg/guide.xml.gz
```

**EPG fallback** (titles only, uncompressed, for players that won't read `.gz`):

```
https://al7omed.github.io/iptv-epg/guide.xml
```

For the EPG to bind to **every channel** in your M3U — including the ~14k that have no `tvg-id` in the original — you also need to patch your local M3U once. See "Patching the M3U" below.

## Why the M3U isn't published

Your IPTV M3U contains stream URLs with embedded auth tokens (`/live/<token>/<token>/<id>.ts`). Hosting that file publicly would leak your subscription. Instead, this repo publishes a small **non-sensitive map**:

```
https://al7omed.github.io/iptv-epg/tvg-id-map.tsv
```

Each row is `tvg-name<TAB>title<TAB>original_tvg_id<TAB>effective_tvg_id`. No URLs, no tokens.

## Patching the M3U

Run `scripts/patch_m3u.py` locally — it downloads the map, injects `tvg-id`s into a copy of your M3U, and writes a new file:

```sh
# one-time: save your original M3U locally
curl -fsSL "<your private M3U URL>" -o ~/Downloads/playlist.m3u

# patch it with auto tvg-ids
python3 scripts/patch_m3u.py ~/Downloads/playlist.m3u ~/Downloads/playlist_patched.m3u
```

Then in your IPTV player:
- **M3U source**: point at the file path you wrote (or move it to your player's storage)
- **EPG source**: `https://al7omed.github.io/iptv-epg/guide.xml.gz`

Re-run the patch script whenever the published map updates (whenever channels are added/removed from your provider's M3U). The mapping is deterministic — same channel name always produces the same `tvg-id`, so the EPG stays bound.

## How it works

The build runs every 12 hours via GitHub Actions:

1. Fetches your M3U (URL stored as `M3U_URL` Secret).
2. Assigns an `effective_id` to every entry — original `tvg-id` if present, else a stable auto-generated id from the channel name.
3. Downloads epgshare01 files: `US2`, `US_LOCALS1`, `US_SPORTS1`, `UK1`, `BEIN1`, `ALJAZEERA1`, `AE1`.
4. Fetches your provider EPG (URL stored as `PROVIDER_EPG_URL` Secret) and includes its channels verbatim — provider has the best display-name alias mappings for ~168 premium UK Sky / AR beIN channels.
5. For each upstream channel, keeps it if it matches an M3U entry (direct tvg-id, name, or US callsign).
6. Dedupes programmes by `(channel, start)` — provider EPG wins on collisions.
7. **Backfill pass**: rewires upstream channel ids to their corresponding M3U `effective_id` so the player actually binds to the real data (otherwise the upstream id like `KNBC-DT.us_locals1` wouldn't match the M3U's `nbc-4-knbc-los-angeles-xxxx.auto`).
8. **Dummy pass**: every remaining `effective_id` not yet covered gets a single 8-day "No EPG" `<programme>` block, snapped to GMT+3 midnight.
9. Writes `docs/guide.xml.gz`, `docs/guide.xml`, and `docs/tvg-id-map.tsv`. Pages serves them.

## Confidence tiers

The pipeline uses sources that themselves aggregate from real-world TV listings (DirecTV, Spectrum, Sky, beIN's own opta API, etc.). Match tiers, ordered by typical accuracy:

| Tier | What | Approx. accuracy |
|---|---|---|
| 1 | Provider EPG (verbatim) | ~100% (provider curated) |
| 2 | beIN MENA via beinsports.com API ([bein-epg repo](https://github.com/al7omed/bein-epg)) | ~95% (source-of-truth) |
| 3 | epgshare01 direct tvg-id match | ~95% |
| 4 | epgshare01 callsign match (e.g. `KNBC` → `KNBC-DT.us_locals1`) | ~80–90% |
| 5 | epgshare01 normalized-name match | ~75–90% |
| 6 | Dummy "No EPG" block | n/a — honest blank |

There is no programmatic 70%-certainty oracle for every channel — TV networks don't all publish open APIs. If you spot a channel with wrong data, add its `effective_tvg_id` (from the map) to `channels/dummy_override.txt` and the build will drop that channel's upstream data and use a dummy instead.

## Manual refresh

```sh
gh workflow run update-epg.yml -R al7omed/iptv-epg
```

## Configuration

Two GitHub Secrets drive the build:

- `M3U_URL` — your M3U playlist URL (with the live-only filter applied).
- `PROVIDER_EPG_URL` — your provider's existing XMLTV URL. Optional but recommended.

Neither is written to disk in the repo or echoed in logs.

## Sister repo

[al7omed/bein-epg](https://github.com/al7omed/bein-epg) — a focused, higher-fidelity EPG for beIN Sports MENA only (24 channels), scraped directly from beinsports.com's opta API. Use both EPGs for best results.
