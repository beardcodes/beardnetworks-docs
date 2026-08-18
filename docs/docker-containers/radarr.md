# Radarr

Radarr is Sonarr for films. Same codebase, same concepts, same settings screens — if you have already set up [Sonarr](sonarr.md), this will take ten minutes. The differences are all in how films behave compared to TV: no episodes to track, but a long gap between a film existing and a good copy existing, which Radarr handles with availability settings.

[Radarr](https://radarr.video/) is a movie collection manager for Usenet and BitTorrent. Version 5 is current.

1. **Monitors films** and grabs them when a release meeting your standards appears
2. **Minimum Availability** — stop it grabbing a cinema cam three months before the digital release
3. **Lists** — import from TMDB collections, IMDb lists, Trakt, or "everything by this director"
4. **Quality profiles and custom formats**, including automatic upgrades
5. **Renames and files** into a layout Plex and Jellyfin understand
6. **Collections** — add one Bond film, get an offer for all of them

## 1. The container

```yaml
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/radarr:/config
      - /mnt/user/data:/data
    ports:
      - 7878:7878
```

```bash
docker compose up -d radarr
```

## 2. Authentication and API key

`http://<host-ip>:7878`. Set **Forms (Login Page)**, then copy the API key from **Settings → General** — Prowlarr, Bazarr and Seerr all need it.

Set **Updates → Mechanism: Docker** while you are on that page.

## 3. Root folder

**Settings → Media Management → Root Folders → Add**: `/data/media/movies`

Container path, not host path. Same rule as Sonarr.

## 4. Media Management

Turn on **Show Advanced Settings**, then:

| Setting | Value |
|---|---|
| Rename Movies | On |
| Replace Illegal Characters | On |
| Use Hardlinks instead of Copy | **On** |
| Import Extra Files | On, `srt,sub` |
| Movie Folder Format | `{Movie CleanTitle} ({Release Year})` |

The folder format matters more for films than for TV. Both Plex and Jellyfin match on title and year; leaving the release group's naming in the folder name causes mismatches, and you end up with a library full of "unknown movie".

The [TRaSH recommended naming scheme](https://trash-guides.info/Radarr/Radarr-recommended-naming-scheme/) is worth copying wholesale for the file format.

## 5. Download clients

**Settings → Download Clients → +**, identical to Sonarr but with the `movies` category:

| Field | qBittorrent | SABnzbd |
|---|---|---|
| Host | `qbittorrent` | `sabnzbd` |
| Port | `8080` | `8080` |
| Auth | WebUI user/pass | API key |
| Category | `movies` | `movies` |

Both use container port `8080` — SABnzbd is only on 8081 from the *host's* point of view.

The category must match what the client has configured, or downloads land in the wrong folder and never import.

## 6. Minimum Availability — the setting that matters most

**Settings → Profiles → Quality Profiles**, or per-film when adding.

This is the one Radarr-specific concept:

| Setting | Meaning | Result |
|---|---|---|
| Announced | A release date exists | Grabs cinema rips and cams. Do not use this. |
| In Cinemas | It is showing in cinemas | Still mostly cams and telesyncs |
| **Released** | Digital or physical release date passed | **What you want** |

Set it to **Released**. Otherwise Radarr dutifully grabs a camcorder recording the week the film opens, marks the film as complete, and stops looking. You then have to notice, delete it, and force a re-search.

!!! note "Released is a date, not a quality check"

    Radarr trusts TMDB's release date. Occasionally TMDB has a digital date that is wrong, and you get an early grab anyway. If a film keeps grabbing rubbish, check its TMDB entry before blaming Radarr.

## 7. Quality profiles

Same logic as Sonarr, but films have a wider size range, so the size limits matter more.

**Settings → Quality** — the max size per quality is defined in MB, not MB/min, for movies. Some sensible caps:

- `WEBDL-1080p`: max around 15 GB
- `Bluray-1080p`: max around 25 GB
- `Bluray-2160p`: leave high if you have the disks, or you will reject every real 4K release

Set the profile cutoff where you stop caring, or Radarr will keep upgrading a film forever.

## 8. Add a film

**Movies → Add New**, search, select.

| Field | Value |
|---|---|
| Root Folder | `/data/media/movies` |
| Minimum Availability | Released |
| Quality Profile | Yours |
| Monitor | Movie Only |

Tick **Start search for missing movie** to grab immediately.

<!-- screenshot: Radarr movie detail page showing file quality and available releases -->

## 9. Import an existing library

If you already have films on disk, **Movies → Import Existing Movies** points at `/data/media/movies` and matches folders to TMDB entries.

Review the matches before confirming. Anything ambiguous gets flagged, and a wrong match here means Radarr will "upgrade" a film by replacing it with a different film entirely.

## 10. Lists, if you want them to arrive automatically

**Settings → Import Lists → +**

TMDB collections, IMDb Top 250, a Trakt watchlist, or another Radarr instance. Each list can auto-add and auto-monitor.

!!! warning "Lists fill disks fast"

    Adding "IMDb Top 250" with monitoring on means Radarr immediately tries to download 250 films. Set **Search on Add** to off for the first sync, review what appeared, then turn it on.

## Updating

```bash
cd /opt/docker/media
docker compose pull radarr
docker compose up -d radarr
```

## Backup

```bash
docker compose stop radarr
sudo tar czf /mnt/user/backups/radarr-$(date +%F).tar.gz -C /mnt/user/appdata radarr
docker compose start radarr
```

Also enable **System → Backup → Scheduled** in the UI.

## Troubleshooting

**Grabbed a cam three months before release.** Minimum Availability is not set to *Released*. Fix it, delete the file, blocklist the release, search again.

**Nothing found for a film that definitely exists.** Use **Manual Search** on the film — it lists every release found and the specific reason each was rejected. Usually a size limit or a quality the profile excludes.

**Downloads complete but do not import.** Category mismatch between Radarr and the client, or permissions. Same debugging as [Sonarr](sonarr.md#troubleshooting).

**Import creates a copy rather than a hardlink.** Volume mount layout. See [the media stack page](media-stack.md#the-one-thing-everyone-gets-wrong-hardlinks).

**Plex shows the wrong film, or "unknown".** Folder naming. It needs `Title (Year)`. Fix the format, then **Movies → Select All → Rename**.

**A film keeps re-downloading.** No cutoff on the profile, so every marginally better release triggers an upgrade. Set a cutoff.

**"Movie already imported" but the file is gone.** Radarr's database and the disk have diverged — usually after a manual delete. **Movies → Select → Refresh** re-scans.

## Where this sits in my lab

Radarr runs alongside Sonarr on the Unraid box, same download clients, same Prowlarr indexers, different category. Requests come in through [Seerr](seerr.md).

The only setting I have changed since setting it up is the size caps. The default profiles happily accept a 60 GB remux of a film I will watch once, and disks are the constraint in this lab.
