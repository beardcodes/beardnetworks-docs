# SABnzbd

SABnzbd is the Usenet half of the download layer. Where torrents give you a swarm and a ratio to maintain, Usenet gives you a direct download at whatever your connection can take, and nothing to seed afterwards.

Set this up **before** Sonarr and Radarr, alongside [qBittorrent](qbittorrent.md).

[SABnzbd](https://sabnzbd.org/) is a Usenet binary downloader that handles the whole pipeline: fetching articles, verifying with PAR2, repairing damaged files, and unpacking the result.

1. **Fully automatic** — hand it an NZB, get a finished file
2. **PAR2 repair** so incomplete downloads recover instead of failing
3. **Categories** with per-category folders and post-processing
4. **Multiple servers** with priorities, so a cheap block account backfills a good unlimited one
5. **Scheduling and bandwidth limits**
6. **A proper API**, which is how the *arr apps drive it

## What you need beyond the container

Usenet is not free, and this is the bit that surprises people coming from torrents. You need two things:

- **A Usenet provider** — the servers holding the actual data. Sold as unlimited monthly, or as blocks of data with no expiry. Retention (how far back the archive goes) and completion rate are what you are paying for.
- **An indexer** — the search layer that produces NZB files. Configured in [Prowlarr](prowlarr.md), not here.

Both are needed. A provider without an indexer means you have nothing to search; an indexer without a provider means you have NZBs pointing at data you cannot fetch.

## 1. The container

```yaml
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
```

**The port mapping is `8081:8080`.** SABnzbd and qBittorrent both use 8080 internally. Change the host side; leave the container side alone, because everything else in the stack talks to it on `8080` over the Docker network.

```bash
docker compose up -d sabnzbd
```

## 2. The setup wizard

`http://<host-ip>:8081`.

The wizard asks for your Usenet provider details. From your provider's welcome email:

| Field | Typical value |
|---|---|
| Host | `news.provider.com` |
| Port | `563` |
| SSL | **On** |
| Username / Password | Your provider credentials |
| Connections | 20 to 50 — check what your plan allows |

Use SSL. Port 563 is the encrypted one; 119 is plaintext. There is no good reason to use 119 in 2026.

**Connections** is the one people get wrong in both directions. Too few and you never saturate your line; more than your provider permits and they start refusing you. Their documentation states the limit — use it.

Test the server before continuing. A green result means the credentials work.

## 3. Set the folders

**Config → Folders**:

| Setting | Value |
|---|---|
| Temporary Download Folder | `/data/usenet/incomplete` |
| Completed Download Folder | `/data/usenet/complete` |

Container paths. These match the [shared layout](media-stack.md#the-folder-layout) so that Sonarr and Radarr can hardlink out of `complete` into the library.

!!! note "Incomplete on fast storage, if you can"

    SABnzbd writes every article to the incomplete folder, then repairs and unpacks. That is a lot of small random I/O. On Unraid, pointing incomplete at a cache pool rather than the array makes a large difference to throughput and to array wear.

    If you do this, incomplete and complete end up on different filesystems, so the final move is a copy rather than a rename. That is fine — the hardlink that matters is the *arr apps' one, from `complete` into `media`, and both of those are on the array.

## 4. Get the API key

**Config → General → Security**. Copy the **API Key**.

Set a username and password on the same page while you are there. SABnzbd is unauthenticated by default.

## 5. Create categories

**Config → Categories**. These route downloads into per-type folders and must match what Sonarr and Radarr send.

| Category | Folder | Priority |
|---|---|---|
| `tv` | `tv` | Default |
| `movies` | `movies` | Default |
| `music` | `music` | Default |
| `books` | `books` | Default |

Leave the folder as a relative name — SABnzbd resolves it under the completed download folder, giving `/data/usenet/complete/tv`.

Set **Post-Processing** to *Default* and leave the script blank. The *arr apps handle everything after the download; a post-processing script here will fight with them.

<!-- screenshot: SABnzbd categories config with tv and movies folders -->

## 6. Switches worth changing

**Config → Switches**:

| Setting | Value | Why |
|---|---|---|
| Post-Process Only Verified Jobs | On | Do not hand broken files to Sonarr |
| Cleanup List | `.nfo, .sfv, .srr, .txt` | Strips clutter before import |
| Unwanted Extensions | `exe, com, bat, sh` with *Blacklist* | A media download has no business containing an executable |
| Direct Unpack | On | Unpacks while downloading; faster if your disk keeps up |
| Pause Downloading During Post-Processing | Off | Unless the box is CPU-starved |

The unwanted extensions setting is a genuine security control, not housekeeping. Usenet is unmoderated and malware in a "release" is a real thing.

## 7. Add a second server, if you have one

**Config → Servers → Add Server**.

The point of a block account is filling gaps. Set your unlimited provider to **Priority 0** and the block account to **Priority 1**. SABnzbd only touches the block account for articles the primary could not supply, so a 500 GB block lasts a very long time.

Tick **Optional** on the backup server so an outage there does not fail downloads.

## 8. Connect Sonarr and Radarr

In each: **Settings → Download Clients → + → SABnzbd**

| Field | Value |
|---|---|
| Host | `sabnzbd` |
| Port | `8080` |
| API Key | From step 4 |
| Category | `tv` for Sonarr, `movies` for Radarr |

**Port 8080, not 8081.** The 8081 mapping only exists on the host; over the Docker network it is still listening on 8080. This trips up nearly everyone.

Under **Completed Download Handling**, leave **Remove** enabled for Usenet. There is nothing to seed, so leaving finished jobs around just wastes disk.

## 9. Bandwidth limits

**Config → General → Bandwidth limit**. Set it to about 80% of your line speed, or use the scheduler to go full speed overnight and throttle during the day.

Usenet will genuinely saturate a gigabit connection with enough connections, which is impressive right up until someone else in the house tries to join a video call.

## Updating

```bash
cd /opt/docker/media
docker compose pull sabnzbd
docker compose up -d sabnzbd
```

## Backup

```bash
docker compose stop sabnzbd
sudo tar czf /mnt/user/backups/sabnzbd-$(date +%F).tar.gz -C /mnt/user/appdata sabnzbd
docker compose start sabnzbd
```

`sabnzbd.ini` holds your provider credentials and API key. Treat the backup as a secret.

## Troubleshooting

**Cannot connect to news server.** Check the port is 563 with SSL on, and that you have not exceeded your connection limit — providers reject the extra connections rather than queuing them.

**Downloads fail verification and repair.** Poor retention or completion at your provider for that release. A block account with a different backbone usually fixes it. If everything fails, suspect the provider rather than SABnzbd.

**Very slow despite a fast line.** Too few connections. Raise it toward your plan's limit. If it is already high, the bottleneck is probably disk I/O on the incomplete folder — see the note in step 3.

**Downloads complete but Sonarr does not import.** Category mismatch, or the folder is not where Sonarr expects. Check that `/data/usenet/complete/tv` exists and Sonarr can see it.

**"Permission denied" on unpack.** PUID/PGID. `chown -R 99:100 /mnt/user/data/usenet`.

**Port 8081 shows nothing.** Another container took it, or you mapped it backwards. `docker ps` and look at the ports column — it should read `0.0.0.0:8081->8080/tcp`.

**High CPU during unpacking.** Normal — PAR2 repair and unrar are CPU-bound. Turn off Direct Unpack on a weak box, or enable *Pause Downloading During Post-Processing*.

**Everything queued but nothing downloading.** SABnzbd is paused. The button is top right, and it is easy to hit accidentally. Also check the scheduler has not paused it on a rule you forgot about.

## Where this sits in my lab

SABnzbd runs on the Unraid box, incomplete on the cache pool, complete on the array. One unlimited provider and one block account for gap-filling.

It does most of the work in this stack. Usenet is faster than torrents for anything recent, and there is no ratio to maintain, so [qBittorrent](qbittorrent.md) only picks up what Usenet cannot supply — older films, and the occasional thing that never made it to a Usenet indexer at all.
