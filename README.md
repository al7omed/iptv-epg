# iptv-epg

Self-updating unified XMLTV EPG for the IPTV playlist. Merges per-region guides from [epgshare01.online](https://epgshare01.online/epgshare01/) with the provider's own EPG, filters down to channels referenced by the M3U, and republishes on a 12-hour cron.

## EPG URLs

Paste **one** of these into your IPTV player as the EPG / TV-guide source:

```
https://al7omed.github.io/iptv-epg/guide.xml.gz
```

(Recommended — 12 MB, gzipped, includes full programme descriptions.)

```
https://al7omed.github.io/iptv-epg/guide.xml
```

(Fallback — 34 MB uncompressed, titles only. Use only if your player doesn't accept `.gz` URLs.)

## How it works

The Python script in `scripts/build_epg.py`:
1. Fetches the M3U playlist (URL stored as a GitHub Secret).
2. Builds an index of every channel — tvg-ids, normalized display-names, US-station callsigns.
3. Downloads the following XMLTV files from epgshare01.online: `US2`, `US_LOCALS1`, `US_SPORTS1`, `UK1`, `BEIN1`, `ALJAZEERA1`, `AE1`.
4. Fetches the provider's own EPG (URL in a Secret) and includes its channels verbatim — the provider already curated multi-alias display-name mappings for ~168 premium UK Sky / AR beIN channels.
5. For each upstream channel, keeps it only if the M3U references it (by tvg-id, by normalized name match, or — for US locals — by callsign match like `KNBC` → `KNBC-DT.us_locals1`).
6. Dedupes programmes by `(channel, start)` — provider EPG wins when sources collide.
7. Writes two files: `docs/guide.xml.gz` (full data) and `docs/guide.xml` (titles only, fits under Pages' 100 MB limit).
8. Commits to `main`. GitHub Pages serves `docs/`.

## Realistic coverage

The M3U has ~18,000 channels but most are PPV/event slots or "DUMP" placeholders that no upstream source has EPG for. The build produces guides for **~1,500 real channels**:

| Source | Channels added |
|---|---|
| Provider EPG (verbatim) | 168 |
| US_LOCALS1 (US affiliates by callsign) | ~727 |
| US2 (US cable/national) | ~200 |
| US_SPORTS1 (US regional sports) | ~73 |
| UK1 (UK broadcast) | ~187 |
| AE1 (UAE / MENA Arabic) | ~190 |
| ALJAZEERA1 | ~1 |
| BEIN1 | 0 (already in provider EPG) |

Channels in `ALL PPV`, `DUMP`, `8K SPORT ON AIR`, and other RAW event groups remain without EPG — they aren't real scheduled channels.

### Dummy EPG for uncovered channels

For every M3U channel that has a `tvg-id` but no upstream EPG match (and for every entry in `channels/dummy_override.txt`), the build adds a placeholder `<channel>` and a series of 6-hour "No EPG" `<programme>` blocks covering the next 3 days. Players that would otherwise leave the row blank now show a uniform grid.

Channels without any `tvg-id` in the M3U can't be helped this way — the player has nothing to bind against. If you want to fix those too, we'd need to republish a modified M3U with auto-generated tvg-ids.

### Marking a channel as inaccurate

If an upstream EPG source matched the wrong programme data to a channel, add the tvg-id to `channels/dummy_override.txt` (one per line, `#` for comments). The real data is dropped and a dummy is used instead.

## Configuration

Two GitHub Secrets drive the build:

- `M3U_URL` — the user's M3U playlist URL (with the live-only filter applied).
- `PROVIDER_EPG_URL` — the provider's existing XMLTV URL. Optional but recommended; it covers premium UK Sky / AR beIN channels well.

These are never written to disk in the repo or echoed in logs. The Actions runner has access at build time via the env vars.

## Manual refresh

```sh
gh workflow run update-epg.yml -R al7omed/iptv-epg
```

## Adding sources or tweaking matching

- To add another country: edit `EPGSHARE_FILES` at the top of `scripts/build_epg.py`. Check the file exists at `https://epgshare01.online/epgshare01/epg_ripper_<NAME>.xml.gz` first.
- To tweak name normalization: edit `normalize_name()` and `extract_callsign()`. The matching is fuzzy by design — adjust the prefix/suffix strip lists for new naming patterns in the M3U.

Commit changes and the workflow runs automatically (it triggers on changes to `scripts/build_epg.py`).
