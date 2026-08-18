# Bazarr

Sonarr and Radarr grab the video. Bazarr grabs the subtitles — and then keeps grabbing them until it finds a version that is actually in sync, which is the part that makes it worth running rather than doing this by hand.

Set it up **after** Sonarr and Radarr, because it works by reading their libraries.

[Bazarr](https://www.bazarr.media/) is a companion to Sonarr and Radarr that manages subtitles.

1. **Automatic downloads** for anything in your library that is missing subtitles
2. **Many providers at once** — OpenSubtitles, Subscene, Podnapisi, Addic7ed and others, with fallback
3. **Scoring** — rejects subtitles that do not match your release, which is what stops the out-of-sync problem
4. **Sync tools** — shift timing, or auto-sync against the audio track using ffsubsync
5. **Upgrades** — replaces a mediocre subtitle when a better-scoring one appears
6. **Per-language profiles**, including forced-only subtitles for foreign dialogue in an English film

!!! note "Prerequisites"

    - [Sonarr](sonarr.md) and [Radarr](radarr.md) running with libraries populated
    - Accounts at a couple of subtitle providers — OpenSubtitles at minimum
    - The [shared stack](media-stack.md) folder layout

## 1. The container

```yaml
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/bazarr:/config
      - /mnt/user/data:/data
    ports:
      - 6767:6767
```

!!! danger "Bazarr's paths must match Sonarr's and Radarr's exactly"

    Bazarr asks Sonarr where a file is, then goes to that path itself and writes an `.srt` next to it. If Sonarr says `/data/media/tv/Show/file.mkv` and Bazarr has that mounted as `/tv/Show/file.mkv`, it will report "file not found" for your entire library.

    With the single `/data` mount used throughout this stack, all three agree by construction. This is the main reason the layout is worth being strict about.

```bash
docker compose up -d bazarr
```

## 2. First run

`http://<host-ip>:6767`.

**Settings → General → Security**: set an authentication method and credentials. Bazarr ships with none.

## 3. Connect Sonarr

**Settings → Sonarr**, toggle **Enabled**.

| Field | Value |
|---|---|
| Address | `sonarr` |
| Port | `8989` |
| Base URL | `/` |
| API Key | Sonarr's, from its Settings → General |
| SSL | Off |

**Test** must go green before saving. Red here is nearly always the address — use the container name, not an IP.

Under the same page, **Minimum Score** for series defaults to 90. Leave it. Lowering it is how you end up with subtitles for a different cut of the episode.

## 4. Connect Radarr

**Settings → Radarr**, same pattern:

| Field | Value |
|---|---|
| Address | `radarr` |
| Port | `7878` |
| API Key | Radarr's |

Movie minimum score defaults to 70. Also fine.

After saving both, Bazarr does an initial sync. **System → Tasks** shows it working through your library — this takes a while on a large one.

## 5. Add subtitle providers

**Settings → Providers → +**

Worth having:

- **OpenSubtitles.com** — the big one. Needs a free account; the free tier has a daily download cap. A VIP account raises it.
- **Podnapisi** — no account needed, decent European coverage
- **Addic7ed** — good for TV, needs an account, aggressive rate limiting
- **Subf2m**, **Supersubtitles** — useful fallbacks

Add three or four, not fifteen. Every provider is queried on every search, so a long list makes searches slow and gets you rate-limited faster.

!!! warning "OpenSubtitles.com is not opensubtitles.org"

    The provider list contains both. `.org` is the legacy API and is largely dead. Use the `.com` entry, and register at opensubtitles.com specifically.

## 6. Create a language profile

**Settings → Languages → Language Profiles → +**

A profile is a list of languages in priority order plus a cutoff:

| Field | Example |
|---|---|
| Name | `English` |
| Languages | English |
| Cutoff | English |
| Must contain | leave empty |

For foreign-language films where you only want subtitles on the non-English dialogue, add a second language entry with **Forced** ticked. Bazarr treats forced subtitles as a separate track.

Then set it as the default: **Settings → Languages → Default Settings**, enabling it for both Series and Movies so newly added items inherit it automatically. Without this, every new show arrives with no profile and Bazarr silently ignores it.

## 7. Apply the profile to existing media

**Series** or **Movies** in the top nav → select all → **Mass Edit** → assign the language profile.

Bazarr then starts searching for everything missing. Expect a lot of activity for the first hour or so.

<!-- screenshot: Bazarr series list showing subtitle status per language -->

## 8. Turn on auto-sync

**Settings → Subtitles**, with advanced settings shown:

| Setting | Value | Why |
|---|---|---|
| Use embedded subtitles | On | Do not download what the file already contains |
| Automatic subtitles synchronization | On | Runs ffsubsync against the audio |
| Series/Movies score threshold | On, 96 / 86 | Only sync when confident |
| Upgrade previously downloaded subtitles | On | Replaces bad ones when better appear |

Auto-sync is the feature that makes Bazarr worth the disk space. It compares the subtitle timing to the actual audio and shifts it, which fixes the "subtitles are 4 seconds ahead" problem without you noticing it happened.

It is CPU-heavy. On a weak box, leave the threshold high so it only runs on subtitles it is likely to fix.

## 9. Check it worked

Pick an episode Bazarr says it grabbed:

```bash
ls /mnt/user/data/media/tv/Some.Show/Season\ 01/
```

You should see `Episode.mkv` and `Episode.en.srt` side by side. The language suffix is what Plex and Jellyfin use to label the track.

## Updating

```bash
cd /opt/docker/media
docker compose pull bazarr
docker compose up -d bazarr
```

Provider APIs change often and Bazarr updates to match, so this is one worth keeping current. [Diun](diun.md) will tell you when.

## Backup

```bash
docker compose stop bazarr
sudo tar czf /mnt/user/backups/bazarr-$(date +%F).tar.gz -C /mnt/user/appdata bazarr
docker compose start bazarr
```

The database holds provider credentials, language profiles and the history of what it has already tried — losing it means re-searching your whole library.

## Troubleshooting

**"No file found" for everything.** Path mismatch with Sonarr or Radarr. Compare the path Sonarr reports for a file against what Bazarr has mounted. This is the number one Bazarr problem and it is always mounts.

**Provider returns 401 or "unauthorized".** Wrong credentials, or you used opensubtitles.org where you meant .com.

**Downloads stop after a while each day.** Provider daily limit. OpenSubtitles free accounts get a modest quota. Add more providers, or pay for VIP.

**Subtitles download but are out of sync.** Turn on automatic synchronization (step 8). For a one-off, use the sync tool on the subtitle itself from the episode view.

**Nothing is being searched at all.** No language profile assigned. **Mass Edit** the library and check **Settings → Languages → Default Settings** is enabled for new items.

**Wrong language downloaded.** Some providers mislabel. Raise the minimum score, and drop the provider if it keeps happening.

**High CPU constantly.** Auto-sync running on everything. Raise the score threshold, or disable it and sync manually when needed.

**Bazarr sees the series but the episode list is empty.** Sonarr's sync has not completed. **System → Tasks → Sync with Sonarr**, run it manually and watch the log.

## Where this sits in my lab

Bazarr runs on the Unraid box next to Sonarr and Radarr, sharing the same `/data` mount — which, as above, is the only reason it works without path mapping gymnastics.

It is the most "set and forget" thing in the stack. It has been running for months and the only time I open it is when a specific film has no decent subtitle and I want to force a manual search.
