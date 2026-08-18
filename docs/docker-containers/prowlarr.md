# Prowlarr

Before Prowlarr, adding an indexer meant adding it in Sonarr, then again in Radarr, then again in Lidarr and Readarr — four times, four sets of credentials, four places to update when the tracker changed domain. Prowlarr does it once and pushes the config to everything else.

Set this up **before** Sonarr and Radarr. It saves you doing the same work twice.

[Prowlarr](https://prowlarr.com/) is an indexer manager for the *arr family. It handles both torrent trackers and Usenet indexers, and syncs them out to every app that needs them.

1. **One place for every indexer** — add once, sync to Sonarr, Radarr, Lidarr, Readarr
2. **500+ built-in definitions**, so most public and private trackers work without manual config
3. **Replaces Jackett and NZBHydra2** — same job, native *arr integration, no extra translation layer
4. **Manual search across all indexers at once**, with a results grid you can sort by seeders or size
5. **Per-indexer stats** — grab counts, failure rates, response times, so you can see which tracker is actually earning its keep
6. **FlareSolverr support** for indexers hiding behind Cloudflare

!!! note "Prerequisites"

    - The stack from [The Media Stack](media-stack.md) running
    - Accounts at whichever indexers you intend to use
    - Your download clients set up first — [qBittorrent](qbittorrent.md), [SABnzbd](sabnzbd.md)

## 1. The container

From the [shared Compose file](media-stack.md#the-compose-file):

```yaml
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/prowlarr:/config
    ports:
      - 9696:9696
```

Prowlarr is the one app in the stack with **no `/data` mount**. It never touches a media file — it only passes search results and magnet links around. If you find yourself adding a media volume here, something has gone wrong in your thinking.

```bash
docker compose up -d prowlarr
```

## 2. Set authentication

Open `http://<host-ip>:9696`. On first run it demands you configure authentication before anything else.

Pick **Forms (Login Page)** and set a username and password. There is an option for *Disabled for Local Addresses* which is convenient and fine on a trusted LAN.

## 3. Add your download clients

**Settings → Download Clients → +**

Prowlarr needs these for the "grab this manually" button to work, and it syncs them out to the *arr apps too.

For qBittorrent:

| Field | Value |
|---|---|
| Host | `qbittorrent` |
| Port | `8080` |
| Username / Password | Your WebUI credentials |
| Category | leave blank here |

For SABnzbd:

| Field | Value |
|---|---|
| Host | `sabnzbd` |
| Port | `8080` |
| API Key | From SABnzbd's **Config → General** |

!!! warning "Use container names, not IP addresses"

    `qbittorrent`, not `192.168.1.50`. Containers on the same Compose network resolve each other by service name, and that keeps working when the host IP changes or you move the stack. Note the port is the **container's internal** port — `8080` for SABnzbd, even though you published it on 8081.

Hit **Test**, then **Save**.

## 4. Add indexers

**Indexers → Add Indexer**, then search the list.

For a public tracker, usually nothing to configure beyond picking it. For a private tracker you will need either an API key or your session cookie, depending on what it supports — the field labels tell you which.

Things worth setting per indexer:

- **Sync Profile** — leave as Standard until you have a reason
- **Tags** — tag your private trackers, so you can later apply different rules to them
- **Priority** — 1 to 50, lower is preferred. Set your good private trackers to 10 and the public ones to 40, and the *arr apps will prefer the good sources when quality is equal.

Test each indexer after adding. A red X here means it will silently return nothing later.

<!-- screenshot: Prowlarr indexer list showing health status and priorities -->

## 5. Add Sonarr and Radarr as applications

This is the step that makes Prowlarr worth running.

**Settings → Apps → +**, then pick Sonarr.

| Field | Value |
|---|---|
| Sync Level | **Full Sync** |
| Prowlarr Server | `http://prowlarr:9696` |
| Sonarr Server | `http://sonarr:8989` |
| API Key | Sonarr's, from its **Settings → General** |

Repeat for Radarr (`http://radarr:7878`), and for Lidarr (`8686`) and Readarr (`8787`) if you run them.

**Sync Level** explained:

- **Full Sync** — Prowlarr manages the indexer list in the target app. Add an indexer here, it appears there. Delete it here, it disappears there. This is what you want.
- **Add and Remove Only** — syncs the existence of indexers but leaves you to configure them in each app.
- **Disabled** — no sync, which defeats the purpose.

!!! danger "Full Sync overwrites indexers in Sonarr and Radarr"

    If you already have indexers configured in Sonarr, Full Sync will take them over and may remove ones Prowlarr does not know about. On an existing setup, note down what you have first. On a fresh setup, this is exactly the behaviour you want.

Hit **Sync App Indexers** from the app's menu, then check Sonarr's **Settings → Indexers**. Everything should be there, with `(Prowlarr)` in the name.

## 6. Add FlareSolverr, if you need it

Some indexers sit behind Cloudflare's browser check. FlareSolverr is a proxy that solves the challenge.

Add to your Compose file:

```yaml
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    restart: unless-stopped
    environment:
      - LOG_LEVEL=info
      - TZ=Europe/London
    ports:
      - 8191:8191
```

Then in Prowlarr: **Settings → Indexers → + → FlareSolverr**, with Host `http://flaresolverr:8191`, and give it the tag `flaresolverr`.

Now tag each indexer that needs it with `flaresolverr`. Only those indexers route through it — sending everything through FlareSolverr is slow and unnecessary.

## 7. Test the whole chain

**Search** in the top nav. Type a film or episode name, hit search with no indexer filter.

Results from multiple indexers means indexers are working. Click the download icon on one — if it lands in qBittorrent, the download client link works too.

That is the whole pipeline verified before Sonarr has done anything.

## Updating

```bash
cd /opt/docker/media
docker compose pull prowlarr
docker compose up -d prowlarr
```

Set **Settings → General → Updates → Mechanism** to *Docker* so the in-app updater stops offering to break itself.

## Backup

`/mnt/user/appdata/prowlarr` holds `prowlarr.db` with every indexer and credential.

Prowlarr also has **System → Backup** which writes into `/config/Backups`. Turn on the scheduled backup — the retention default is sensible.

```bash
docker compose stop prowlarr
sudo tar czf /mnt/user/backups/prowlarr-$(date +%F).tar.gz -C /mnt/user/appdata prowlarr
docker compose start prowlarr
```

## Troubleshooting

**Indexer test fails with a Cloudflare error.** Add FlareSolverr, step 6.

**Indexer test fails with 401 or 403.** Wrong API key or an expired cookie. Private tracker cookies expire — re-copy from your browser.

**Sync says success but Sonarr has no indexers.** Check the API key, and check Prowlarr can actually reach Sonarr: `docker exec prowlarr curl -s http://sonarr:8989/ping`. If that fails they are not on the same Docker network.

**Searches return nothing from a working indexer.** Usually the categories. Check the indexer's category mappings — a tracker that files everything under a nonstandard category ID returns zero results for a correctly-formed TV search.

**"Query successful but no results" for everything.** Your search terms are being mangled. Try the exact release name. If manual search in the *tracker's own website* works and Prowlarr's does not, the indexer definition may be out of date — update the container.

**Indexer keeps auto-disabling.** Prowlarr disables indexers that fail repeatedly. **System → Events** shows why. Usually rate limiting: you are hammering the tracker with too many *arr apps searching too often.

## Where this sits in my lab

Prowlarr runs on the Unraid box with the rest of the stack. It manages a mix of one Usenet indexer and a handful of trackers, syncing to Sonarr, Radarr, Lidarr and Readarr — four apps, one place to fix things when a tracker moves domain.

The per-indexer stats page is the underrated part. It made it obvious that two of the public trackers I was carrying had a 90% failure rate and were just adding latency to every search. Removing them made the whole stack feel faster.
