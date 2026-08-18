# qBittorrent

qBittorrent is the torrent half of the download layer. Sonarr and Radarr hand it a torrent, it downloads into a category folder, and the *arr apps hardlink the result into the library while the original keeps seeding.

Set this up **before** Sonarr and Radarr, so their download client config has something to point at.

[qBittorrent](https://www.qbittorrent.org/) is an open-source BitTorrent client with a web UI and a proper API. No ads, no crypto miners, no nag screens.

1. **Web UI and full API**, which is how the *arr apps drive it
2. **Categories** — route each download to its own folder automatically
3. **Sequential download** and bandwidth scheduling
4. **RSS with auto-download rules**, if you want to bypass the *arr apps for something
5. **IP filtering and encryption** settings
6. **Seeding limits** by ratio or time, with automatic removal

## 1. The container

```yaml
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
```

The volume mount is deliberately `/data/torrents`, not all of `/data`. qBittorrent only needs its own download tree; Sonarr and Radarr are the ones that need to see both sides to hardlink across.

**`TORRENTING_PORT` must match** the listening port you set in the UI later, and both TCP and UDP need forwarding for good peer connectivity.

```bash
docker compose up -d qbittorrent
```

## 2. Get the temporary password

Since version 4.6.1 there is no default `adminadmin` password. A random one is generated on first start and printed once, to the log:

```bash
docker logs qbittorrent | grep -i "temporary password"
```

Open `http://<host-ip>:8080`, log in as `admin` with that password.

If you missed it or the log rotated, delete `/mnt/user/appdata/qbittorrent/qBittorrent/qBittorrent.conf` and restart the container to get a fresh one.

## 3. Change the password immediately

**Tools → Options → Web UI**:

- Set a real username and password
- Tick **Bypass authentication for clients on localhost** only if you understand the implication
- Leave **Enable clickjacking protection** and **CSRF protection** on

!!! danger "Never expose the qBittorrent WebUI to the internet"

    It has no rate limiting on login, and an attacker with access can set the download path to anywhere the container can write and use it as a file drop. Keep it on the LAN, behind WireGuard, or behind an authenticating reverse proxy — never a plain port forward.

## 4. Set the download paths

**Tools → Options → Downloads**:

| Setting | Value |
|---|---|
| Default Save Path | `/data/torrents` |
| Keep incomplete torrents in | On, `/data/torrents/incomplete` |
| Copy .torrent files for finished downloads | Off |
| Pre-allocate disk space for all files | On |
| Append .!qB extension to incomplete files | On |

Pre-allocation reduces fragmentation on the array. The `.!qB` extension stops the *arr apps trying to import a file that is still downloading.

## 5. Create categories

This is the step that makes the whole stack work. **Right-click in the Categories sidebar → Add category**:

| Category | Save path |
|---|---|
| `tv` | `/data/torrents/tv` |
| `movies` | `/data/torrents/movies` |
| `music` | `/data/torrents/music` |
| `books` | `/data/torrents/books` |

The category names must match **exactly** what you enter in Sonarr and Radarr's download client settings. `tv` and `TV` are different categories, and a mismatch means the download completes and then nothing happens forever.

<!-- screenshot: qBittorrent categories sidebar with save paths configured -->

## 6. Seeding limits

**Tools → Options → BitTorrent**, at the bottom:

| Setting | Suggested |
|---|---|
| When ratio reaches | 2.0 |
| When seeding time reaches | 20160 minutes (14 days) |
| Then | **Pause torrent** |

Choose **Pause**, not **Remove**. Removing the torrent can delete the file, and because the *arr apps hardlinked it, deleting the original is usually harmless — but "usually" is doing a lot of work in that sentence, and a paused torrent costs nothing.

If you use private trackers, check their rules first. Many require a minimum seed time and will penalise you for stopping early.

## 7. Connection settings

**Tools → Options → Connection**:

- **Port used for incoming connections**: `6881`, matching `TORRENTING_PORT`
- **Use UPnP / NAT-PMP**: off — forward the port manually at the router instead, or leave it closed if you are behind a VPN
- **Global maximum connections**: 500 is plenty; higher just adds CPU load

Under **Speed**, set a global upload cap at around 80% of your actual upload bandwidth. Saturating the uplink makes everything else in the house feel broken, and it is the most common "my internet is slow" cause in a homelab.

## 8. Connect Sonarr and Radarr

In each app: **Settings → Download Clients → + → qBittorrent**

| Field | Value |
|---|---|
| Host | `qbittorrent` |
| Port | `8080` |
| Username / Password | What you set in step 3 |
| Category | `tv` for Sonarr, `movies` for Radarr |

Container name and internal port — not the host IP.

## Running it behind a VPN

If you want the torrent traffic tunnelled, the pattern is [gluetun](https://github.com/qdm12/gluetun) as a network namespace:

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
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
    environment:
      - PUID=99
      - PGID=100
      - UMASK=022
      - TZ=Europe/London
      - WEBUI_PORT=8080
    volumes:
      - /mnt/user/appdata/qbittorrent:/config
      - /mnt/user/data/torrents:/data/torrents
    restart: unless-stopped
```

Note what changed: qBittorrent has **no `ports:` block at all**, because it has no network stack — gluetun publishes on its behalf. Sonarr and Radarr must then point at `gluetun`, not `qbittorrent`, since that is where the port lives.

Verify the tunnel actually works before trusting it:

```bash
docker exec qbittorrent curl -s https://ipinfo.io/ip
```

That should return the VPN's exit IP, not yours. If it returns your home IP, the tunnel is not up and you should stop downloading until it is.

## Updating

```bash
cd /opt/docker/media
docker compose pull qbittorrent
docker compose up -d qbittorrent
```

!!! warning "qBittorrent occasionally changes the config format"

    Major version jumps have historically reset WebUI settings or changed defaults. Back up `/config` before a big upgrade, and read the LinuxServer changelog if the tag has moved a whole version.

## Backup

```bash
docker compose stop qbittorrent
sudo tar czf /mnt/user/backups/qbittorrent-$(date +%F).tar.gz -C /mnt/user/appdata qbittorrent
docker compose start qbittorrent
```

`/config` includes `BT_backup`, which holds the `.torrent` and `.fastresume` files for everything you are seeding. Lose that and you lose your seeding history — which on a private tracker is a real cost.

## Troubleshooting

**Cannot log in on a fresh install.** Temporary password in the logs, step 2.

**Torrents stuck at "stalled".** No incoming connections. Check the port is forwarded, or if you are on a VPN, that the provider supports port forwarding — many do not, and downloads still work but slowly.

**Downloads finish but Sonarr never imports.** Category mismatch, nine times out of ten. Compare the category string in qBittorrent against the one in Sonarr's download client settings, character for character.

**"Permission denied" writing to the download folder.** PUID/PGID. `chown -R 99:100 /mnt/user/data/torrents`.

**WebUI unreachable after adding gluetun.** Expected — the port moved to the gluetun container. Reach it at gluetun's address, and update the *arr apps to point at `gluetun`.

**Everything is slow and the array is thrashing.** Torrents seeding directly from the array. Either accept it, or move the incomplete folder to a cache pool so only completed files touch the disks.

**Ratio is stuck at 0 on a private tracker.** The announce is failing. Check the tracker tab on a torrent for the error — usually an expired passkey after you re-added the indexer in Prowlarr.

## Where this sits in my lab

qBittorrent runs on the Unraid box with the rest of the stack, categories matching the folder layout, and a global upload cap so nobody in the house notices it exists.

It handles the releases that Usenet does not have — mostly older films and anything niche. [SABnzbd](sabnzbd.md) does the heavy lifting for everything current, because Usenet is faster and does not need seeding.
