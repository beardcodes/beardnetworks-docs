# Sonarr

Sonarr is the brain of the TV side of the stack. You tell it which series you care about; it watches for episodes, asks the indexers, hands the best result to a download client, and then renames and files everything so your media server can make sense of it.

This guide assumes [The Media Stack](media-stack.md) is running and [Prowlarr](prowlarr.md) has synced its indexers.

[Sonarr](https://sonarr.tv/) is a PVR for Usenet and BitTorrent. Version 4 is current.

1. **Monitors series and seasons**, grabs new episodes as they air
2. **Quality profiles** — define what "good enough" means, and it upgrades automatically when something better appears
3. **Renames and organises** into a consistent structure Plex and Jellyfin can parse
4. **Handles the messy parts** of TV: multi-episode files, season packs, absolute numbering for anime, specials
5. **Failed download handling** — a release that does not import gets blocklisted and replaced automatically
6. **Calendar and iCal feed** so you know what is coming

## 1. The container

```yaml
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/sonarr:/config
      - /mnt/user/data:/data
    ports:
      - 8989:8989
```

The single `/data` mount is what makes hardlinks and instant imports work. If you skipped [that section](media-stack.md#the-one-thing-everyone-gets-wrong-hardlinks), go back — this is the decision that costs you a weekend later.

```bash
docker compose up -d sonarr
```

## 2. Authentication

`http://<host-ip>:8989`. Set **Forms (Login Page)** with a username and password.

Grab your API key now from **Settings → General** — Prowlarr, Bazarr and Seerr will all want it.

## 3. Set the update mechanism

**Settings → General → Updates → Mechanism: Docker**.

Do this before you forget. Sonarr's built-in updater does not work sensibly in a container — it either fails, or succeeds and gets wiped by your next `docker compose pull`.

## 4. Add the root folder

**Settings → Media Management → Root Folders → Add Root Folder**

Set it to `/data/media/tv`.

!!! danger "The root folder must be inside `/data`"

    Not `/tv`. Not `/mnt/user/data/media/tv`. The path as **the container sees it**, which with the standard mount is `/data/media/tv`. Get this wrong and imports either fail or copy instead of hardlink.

## 5. Media Management settings

Same page, and these are the ones that matter:

| Setting | Value | Why |
|---|---|---|
| Rename Episodes | On | Otherwise you keep the release group's naming |
| Replace Illegal Characters | On | Windows shares will thank you |
| Use Hardlinks instead of Copy | **On** | The whole point |
| Import Extra Files | On, `srt,sub` | Picks up bundled subtitles |
| Unmonitor Deleted Episodes | On | Stops it re-downloading things you deleted on purpose |

Turn on **Show Advanced Settings** (the toggle at the top) to see the hardlink option.

For naming, the [TRaSH Guides recommended format](https://trash-guides.info/Sonarr/Sonarr-recommended-naming-scheme/) is worth pasting in rather than inventing your own. Both Plex and Jellyfin parse it correctly, which is not true of every scheme.

## 6. Connect the download clients

**Settings → Download Clients → +**

qBittorrent:

| Field | Value |
|---|---|
| Host | `qbittorrent` |
| Port | `8080` |
| Username / Password | Your WebUI login |
| Category | `tv` |

SABnzbd:

| Field | Value |
|---|---|
| Host | `sabnzbd` |
| Port | `8080` |
| API Key | From SABnzbd |
| Category | `tv` |

The **category** is what routes downloads into `/data/torrents/tv` and `/data/usenet/complete/tv`. It must match the category you configured in the client itself. Mismatched categories are the single most common cause of "downloaded but never imported".

Under **Settings → Download Clients → Completed Download Handling**, leave **Remove** on for Usenet, and off for torrents if you care about seeding ratios.

## 7. Check the indexers arrived

**Settings → Indexers** should already be populated by Prowlarr, each name suffixed `(Prowlarr)`.

Empty? Go back to Prowlarr, **Settings → Apps**, and hit **Sync App Indexers**. Do not add indexers here manually — Full Sync will remove them.

## 8. Quality profiles

**Settings → Profiles**. The defaults work, but two adjustments make a real difference:

**Set a sensible cutoff.** A profile with `HDTV-720p` through `Bluray-2160p` all enabled and no cutoff will keep upgrading forever, re-downloading the same episode at increasing sizes. Set the cutoff to where you actually stop caring — usually `WEBDL-1080p`.

**Add a size limit.** **Settings → Quality** lets you set min and max MB per minute per quality. Capping `WEBDL-1080p` at around 100 MB/min keeps 40 GB "1080p" remuxes out of your library.

If you want to go further, [TRaSH's custom formats](https://trash-guides.info/Sonarr/sonarr-collection-of-custom-formats/) let you prefer specific release groups and reject bad encodes. That is a rabbit hole; the defaults are fine to start.

## 9. Add a series

**Series → Add New**, search, pick it.

| Field | Note |
|---|---|
| Root Folder | `/data/media/tv` |
| Monitor | *All* for a finished show, *Future Episodes* for one still airing |
| Quality Profile | Whichever you configured |
| Series Type | *Standard*, or *Anime* if it uses absolute numbering |
| Season Folder | On |

Tick **Start search for missing episodes** and it goes to work immediately.

<!-- screenshot: Sonarr series view with a season expanded showing episode status -->

Watch **Activity → Queue** to see the grab, then **Activity → History** for the import. A successful import shows the file moving from `/data/torrents/tv/...` to `/data/media/tv/...`.

## 10. Verify the hardlink

The first import is the one to check:

```bash
ls -la /mnt/user/data/media/tv/Some.Show/Season\ 01/
```

Link count of `2` in the second column means hardlinked. `1` means Sonarr copied it, your disk usage just doubled, and something is wrong with your mounts.

## Updating

```bash
cd /opt/docker/media
docker compose pull sonarr
docker compose up -d sonarr
```

Sonarr migrates its database on start. Watch the logs the first time after a major version change:

```bash
docker compose logs -f sonarr
```

## Backup

`/mnt/user/appdata/sonarr` holds `sonarr.db` — your entire series list, history and settings.

Turn on **System → Backup → Scheduled** in the UI, then include the appdata folder in your host-level backup:

```bash
docker compose stop sonarr
sudo tar czf /mnt/user/backups/sonarr-$(date +%F).tar.gz -C /mnt/user/appdata sonarr
docker compose start sonarr
```

Stop it first. A SQLite database copied mid-write restores as a corrupt database, and you will not find out until you need it.

## Troubleshooting

**Download completes but never imports.** In order of likelihood: category mismatch between Sonarr and the client; the download client reporting a path Sonarr cannot see; permissions. Check **Activity → Queue** — Sonarr usually states the reason on hover.

**"Permission denied" on import.** PUID/PGID. `ls -la` the downloaded file and compare against Sonarr's env vars. `chown -R 99:100 /mnt/user/data`.

**Imports are copies, not hardlinks.** Multiple volume mounts, or downloads and media on different Unraid shares. See [the layout section](media-stack.md#the-folder-layout).

**Nothing is found for a show that definitely has releases.** Check **Manual Search** on a specific episode — it lists every release and, crucially, the reason each was rejected. Usually a quality profile that excludes everything available, or a size limit that is too tight.

**Anime episode numbers are wrong.** Set Series Type to *Anime*. Sonarr then uses absolute numbering and consults AniDB mappings.

**Same episode downloading over and over.** No cutoff on the quality profile, so every slightly-better release triggers an upgrade. Set one.

**Series added but the calendar is empty.** TheTVDB has not got the schedule, or the series is unmonitored. **Series → Edit → Monitor** to check.

**Sonarr is very slow on Unraid.** `appdata` on the array rather than the cache pool. Move it.

## Where this sits in my lab

Sonarr runs on the Unraid box with the rest of the stack, pulling from both Usenet and torrents through Prowlarr, and handing off to SABnzbd or qBittorrent depending on where the release came from.

Requests arrive from [Seerr](seerr.md), so the people who watch the TV never see this interface — which is the correct arrangement, because the quality profile screen is not a thing anyone else should have opinions about.
