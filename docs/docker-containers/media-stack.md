# The Media Stack

Read this page before the individual app guides. Nine containers that each work fine alone become a coherent system only if you get two things right at the start: **the folder layout** and **the user IDs**. Get them wrong and everything still appears to work, while quietly doubling your disk usage and breaking permissions in ways that take a weekend to unpick.

This is the shared foundation. Each app then gets its own page for the parts specific to it.

## What the stack does

| App | Job | Port |
|---|---|---|
| [Prowlarr](prowlarr.md) | Indexer manager — one place for all trackers and Usenet indexers | 9696 |
| [Sonarr](sonarr.md) | TV — monitors series, requests episodes, renames and files them | 8989 |
| [Radarr](radarr.md) | Films — same, for movies | 7878 |
| [Bazarr](bazarr.md) | Subtitles for whatever Sonarr and Radarr have grabbed | 6767 |
| [qBittorrent](qbittorrent.md) | Torrent download client | 8080 |
| [SABnzbd](sabnzbd.md) | Usenet download client | 8081 |
| [Seerr](seerr.md) | Request front end for people who should not have Sonarr logins | 5055 |
| Lidarr | Music | 8686 |
| Readarr | Books and audiobooks | 8787 |

The flow: someone requests a film in Seerr → Seerr tells Radarr → Radarr asks Prowlarr for releases → Prowlarr returns results from every indexer → Radarr picks one and hands it to qBittorrent or SABnzbd → the client downloads it → Radarr imports, renames and files it → Bazarr fetches subtitles → Plex or Jellyfin serves it.

## The one thing everyone gets wrong: hardlinks

Nearly every guide tells you to mount `/tv`, `/movies` and `/downloads` as three separate volumes. Do not do this.

A hardlink is a second name for the same data on disk. When Radarr "moves" a finished download into your library, it can either **hardlink** (instant, uses no extra space, the torrent keeps seeding) or **copy** (slow, doubles the space, and now you have two copies of a 60 GB remux).

Hardlinks only work **within a single filesystem**. Docker treats each volume mount as its own filesystem boundary. So:

```yaml
# WRONG - three mounts, no hardlinks possible
volumes:
  - /mnt/user/media/tv:/tv
  - /mnt/user/media/movies:/movies
  - /mnt/user/downloads:/downloads
```

```yaml
# RIGHT - one mount, hardlinks and atomic moves work
volumes:
  - /mnt/user/data:/data
```

With the wrong layout, every import is a full copy. Your array fills up twice as fast and imports take minutes instead of milliseconds.

!!! danger "On Unraid, this also means one share"

    `/mnt/user/data` must be a single share. If `downloads` and `media` are separate Unraid shares, they can land on different disks — different filesystems — and hardlinks fail even though the Docker mount looks correct. One `data` share, subfolders inside it.

## The folder layout

Create this once, inside a single share:

```bash
mkdir -p /mnt/user/data/torrents/{books,movies,music,tv}
mkdir -p /mnt/user/data/usenet/{incomplete,complete}
mkdir -p /mnt/user/data/media/{books,movies,music,tv}
```

Which gives you:

```
/mnt/user/data
├── torrents          <- qBittorrent writes here, by category
│   ├── books
│   ├── movies
│   ├── music
│   └── tv
├── usenet            <- SABnzbd writes here
│   ├── incomplete
│   └── complete
└── media             <- the library Plex and Jellyfin read
    ├── books
    ├── movies
    ├── music
    └── tv
```

Every container in the stack mounts `/mnt/user/data:/data` and nothing else. Every app sees identical paths, which also means Bazarr, Sonarr and Radarr agree about where a file is — the other common source of "works but cannot find the file" bugs.

This layout comes from the [TRaSH Guides](https://trash-guides.info/File-and-Folder-Structure/), which are worth reading in full if you plan to take this seriously.

## PUID, PGID and UMASK

LinuxServer.io images run as whatever user you tell them to. All of them must agree, or Sonarr will write files Bazarr cannot read.

**On Unraid**, the answer is always:

```yaml
environment:
  - PUID=99      # nobody
  - PGID=100     # users
  - UMASK=022
```

**On a normal Linux host**, use your own IDs:

```bash
id $USER
# uid=1000(user) gid=1000(user)
```

`UMASK=022` gives new files `644` and folders `755` — readable by the group. `002` if you need group-writable.

Fix any existing mess before you start:

```bash
sudo chown -R 99:100 /mnt/user/data
sudo chmod -R 775 /mnt/user/data
```

## The Compose file

One file, one stack. Split into separate stacks only if you have a reason to.

```bash
sudo mkdir -p /opt/docker/media && cd /opt/docker/media
sudo nano docker-compose.yml
```

```yaml
services:
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

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /mnt/user/appdata/qbittorrent:/config
      - /mnt/user/data/torrents:/data/torrents
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp

  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    restart: unless-stopped
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
    volumes:
      - /mnt/user/appdata/sabnzbd:/config
      - /mnt/user/data/usenet:/data/usenet
    ports:
      - 8081:8080

  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    init: true
    restart: unless-stopped
    environment:
      - LOG_LEVEL=info
      - TZ=Europe/London
      - PORT=5055
    volumes:
      - /mnt/user/appdata/seerr:/app/config
    ports:
      - 5055:5055
```

Notes on the choices:

1. **`appdata` is separate from `data`.** Configs and databases live on the cache/SSD; media lives on the array. Two different backup strategies, two different performance needs.
2. **SABnzbd is `8081:8080`.** Both it and qBittorrent default to 8080 internally. Change the *host* side, not the container side.
3. **The download clients mount a subpath.** qBittorrent gets `/data/torrents`, SABnzbd gets `/data/usenet`. They only need their own tree. Sonarr and Radarr get all of `/data` because they read from downloads and write to media — that is where the hardlink happens.
4. **Seerr is not a LinuxServer image**, so no PUID/PGID. It uses `init: true` instead.
5. **`TZ` matters.** Wrong timezone means schedules, RSS syncs and log timestamps are all offset, which makes debugging miserable.

Start it:

```bash
docker compose up -d
docker compose ps
```

## Setup order

The apps reference each other, so sequence matters. Do it in this order and nothing needs revisiting:

1. **[qBittorrent](qbittorrent.md)** and **[SABnzbd](sabnzbd.md)** — get the download clients working and their categories defined first.
2. **[Prowlarr](prowlarr.md)** — add indexers, then add Sonarr and Radarr as applications so indexers sync automatically.
3. **[Sonarr](sonarr.md)** and **[Radarr](radarr.md)** — root folders, then connect the download clients.
4. **[Bazarr](bazarr.md)** — connect it to Sonarr and Radarr.
5. **[Seerr](seerr.md)** — connect it to Plex or Jellyfin, then to Sonarr and Radarr.

Every app has an API key under **Settings → General**. You will be copying those between apps constantly, so open them all in tabs.

## Verifying hardlinks actually work

Do not assume. After your first successful import:

```bash
# Find the file in your library, note the link count (second column)
ls -la /mnt/user/data/media/movies/Some.Movie.2024/
```

A hardlinked file shows `2` where a normal file shows `1`. Or compare inode numbers directly:

```bash
find /mnt/user/data -inum $(stat -c %i "/mnt/user/data/media/movies/Some.Movie.2024/Some.Movie.2024.mkv")
```

Two paths returned — one in `torrents`, one in `media` — means it worked. One path means Radarr copied the file, and something about your mounts is wrong.

Also check disk usage: `du -sh` on the whole `data` share should be roughly the size of your library, not double it.

## Should the download client go through a VPN?

If you torrent, probably. The usual approach is [gluetun](https://github.com/qdm12/gluetun) as a network gateway:

```yaml
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=mullvad
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WIREGUARD_ADDRESSES}
      - SERVER_CITIES=Amsterdam
    ports:
      - 8080:8080        # qBittorrent WebUI, published here instead
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped

  qbittorrent:
    # ... as above, but:
    network_mode: "service:gluetun"
    # and remove the entire ports: block
```

When a container uses `network_mode: service:gluetun`, it has no network stack of its own — so its ports must be published on gluetun instead, and the WebUI is reached at the gluetun container's address.

!!! warning "Only the download client goes behind the VPN"

    Do not put Sonarr, Radarr or Prowlarr behind it. They need to reach your indexers and each other, many indexers block VPN exit IPs, and a VPN dropout then takes down the whole stack rather than just pausing downloads. Gluetun's killswitch is the point: if the tunnel dies, qBittorrent loses all connectivity, which is exactly what you want.

## Updating

```bash
cd /opt/docker/media
docker compose pull
docker compose up -d
docker image prune -f
```

The *arr apps handle their own database migrations on start. Watch the logs the first time after a major version bump.

[Diun](diun.md) will notify you when new images land, which beats `:latest` silently changing under you at 3am.

!!! note "Do not update the *arr apps from inside their own UI"

    Sonarr and Radarr have a built-in updater. In a container it either fails or produces an install that your next `docker compose pull` overwrites. Set **Settings → General → Updates → Mechanism** to *Docker* so the app stops offering.

## Backup

`appdata` is what matters — the databases hold your entire library history, quality profiles and indexer config. The media is replaceable; the config is the part you would hate to rebuild.

```bash
cd /opt/docker/media
docker compose stop
sudo tar czf /mnt/user/backups/media-appdata-$(date +%F).tar.gz \
  -C /mnt/user/appdata prowlarr sonarr radarr bazarr qbittorrent sabnzbd seerr
docker compose start
```

Stop the containers first — SQLite databases copied mid-write restore as corrupt.

Sonarr and Radarr also produce their own backups under **System → Backup**, which land in `/config/Backups` and get included by the above.

## Troubleshooting

**Imports fail: "Permission denied".** PUID/PGID mismatch. `ls -la` the file and compare to your env vars. Then `chown -R 99:100 /mnt/user/data`.

**Every import is a copy, not a hardlink.** Your mounts are wrong. Back to the folder layout section — you almost certainly have more than one volume mount, or downloads and media are on different Unraid shares.

**"Download client reports the file exists but Sonarr cannot see it".** Path mismatch. The download client is telling Sonarr about a path from its own perspective. With the single `/data` mount, both see the same path and this disappears. If you inherited a different layout, use **Remote Path Mappings** in Sonarr as a workaround, but fixing the mounts is the real answer.

**Two containers both try to bind port 8080.** SABnzbd and qBittorrent. Remap the host side.

**qBittorrent WebUI rejects your password on a fresh install.** Since 4.6.1 it generates a random one. `docker logs qbittorrent | grep -i password`.

**Everything is slow on Unraid.** `appdata` is on the array instead of the cache pool. SQLite on spinning disks through a fuse filesystem is painful. Move it to cache and set the share to *Prefer: cache*.

## Where this sits in my lab

The whole stack runs on the Unraid box, because that is where the disks are. Unraid handles the array and the shares; Docker handles everything above that. Plex and Jellyfin both read `/mnt/user/data/media` — Plex for the people I share with, Jellyfin because I prefer it and it does not need an account to work.

Requests come in through Seerr, which is the only part of this that anyone else sees. Nobody but me has a Sonarr login, which is exactly how it should be.

The Proxmox nodes deliberately run none of this. Media is disk-bound and the hypervisors are not where the disks live.
