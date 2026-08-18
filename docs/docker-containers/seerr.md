# Seerr (Jellyseerr)

Everything else in the media stack is an admin tool. Seerr is the part other people see: a clean search page where someone asks for a film, and it appears a while later without them ever knowing Radarr exists.

Set it up **last**, once Sonarr, Radarr and your media server are all working.

!!! note "Jellyseerr is now Seerr"

    The project was renamed. The current image is `ghcr.io/seerr-team/seerr`, and the docs moved to [docs.seerr.dev](https://docs.seerr.dev/). Existing `fallenbagel/jellyseerr` installs still run, but new deployments should use the new image — this guide uses it throughout.

[Seerr](https://docs.seerr.dev/) is a request management and media discovery tool. It is a fork of Overseerr that added Jellyfin and Emby support alongside Plex.

1. **Request portal** for films and TV, with search across TMDB
2. **Works with Plex, Jellyfin and Emby**, importing users from any of them
3. **Approval workflow** — auto-approve people you trust, hold requests from those you do not
4. **Quotas** per user, so nobody adds 300 films on a Sunday afternoon
5. **Partial season requests** — someone can ask for one season rather than a whole series
6. **Notifications** to Discord, ntfy, email, webhooks, and to the requester when their thing is ready

## 1. The container

Not a LinuxServer image, so no PUID/PGID:

```yaml
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
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/settings/public || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
```

No `/data` mount — Seerr never touches a media file. It talks to APIs only.

Upstream ships `LOG_LEVEL=debug`; `info` is saner for something you intend to leave running.

```bash
docker compose up -d seerr
```

## 2. Sign in with your media server

`http://<host-ip>:5055`. The first screen asks which media server you use.

**For Jellyfin:** enter the internal URL `http://jellyfin:8096` and sign in with a Jellyfin **admin** account. That account becomes the Seerr owner.

**For Plex:** click **Sign in with Plex**, authenticate, then pick your server from the list.

**Running both, like I do?** Pick one as the auth source. Seerr supports one media server at a time, so choose whichever one the people making requests actually use — for most homelabs that is Plex, because that is who you share with.

## 3. Configure libraries

After connecting, tick the libraries Seerr should scan and hit **Start Scan**.

This is how it knows what you already have. Without it, everything shows as "Request" even when the film is sitting in your library, and you get duplicate requests.

The scan takes a while on a large library. It runs on a schedule afterwards.

## 4. Connect Radarr

**Settings → Services → Add Radarr Server**

| Field | Value |
|---|---|
| Default Server | On |
| Server Name | `Radarr` |
| Hostname | `radarr` |
| Port | `7878` |
| API Key | Radarr's |
| Quality Profile | Your default |
| Root Folder | `/data/media/movies` |
| Minimum Availability | Released |
| Enable Scan | On |
| Enable Automatic Search | On |

**Test** must succeed before the save button works.

The root folder dropdown pulls live from Radarr. If it is empty, Radarr has no root folder configured — go back to [Radarr](radarr.md#3-root-folder).

## 5. Connect Sonarr

**Settings → Services → Add Sonarr Server**

| Field | Value |
|---|---|
| Default Server | On |
| Hostname | `sonarr` |
| Port | `8989` |
| API Key | Sonarr's |
| Quality Profile | Your default |
| Root Folder | `/data/media/tv` |
| Season Folders | On |
| Enable Automatic Search | On |

If you keep anime separate in Sonarr with its own profile and root folder, add a **second** Sonarr entry and tick **Anime** on it. Seerr routes anime requests there automatically.

## 6. Add users

**Settings → Users → Import Users** pulls in accounts from Plex or Jellyfin.

Then, per user or via **Settings → Users → General** as a default:

| Permission | Suggestion |
|---|---|
| Request | On |
| Auto-Approve | On for people you trust, off otherwise |
| Request Movies / Series | On |
| Advanced Requests | **Off** — do not let them pick quality profiles |
| Manage Requests | Off, that is your job |

**Set quotas.** `Settings → Users → General → Global Movie Request Limit`, e.g. 10 per week. Without one, somebody discovers the search box and adds a director's entire filmography, and your array fills overnight. This is not hypothetical.

<!-- screenshot: Seerr discover page with pending and available badges -->

## 7. Notifications

**Settings → Notifications**. Two worth configuring:

**For you** — a Discord webhook or ntfy topic for *Request Pending Approval*, so you know when something needs a decision.

**For them** — enable the **Request Available** notification so requesters get told when their film is ready. This single feature stops the "is it there yet" messages entirely, and is the main reason to run Seerr rather than just handing people a Sonarr login.

Test each agent before trusting it.

## 8. Put it behind a reverse proxy

Seerr is the one part of this stack that might legitimately face the internet, since the people requesting are not always on your network.

```yaml
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
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.seerr.rule=Host(`requests.example.com`)"
      - "traefik.http.routers.seerr.entrypoints=https"
      - "traefik.http.routers.seerr.tls.certresolver=letsencrypt"
      - "traefik.http.services.seerr.loadbalancer.server.port=5055"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Then set **Settings → General → Application URL** to `https://requests.example.com`, or the links in notification emails point at an internal address nobody outside can reach.

!!! warning "Exposing Seerr exposes an account system"

    If you publish this, you are publishing a login page tied to your Plex or Jellyfin accounts. Turn off **Enable Local Sign-In** if you only want media-server accounts, keep the app updated, and consider putting [CrowdSec](traefik.md) or a geo-block in front of it. The rest of the stack should stay firmly on the LAN.

## Updating

```bash
cd /opt/docker/media
docker compose pull seerr
docker compose up -d seerr
```

## Backup

```bash
docker compose stop seerr
sudo tar czf /mnt/user/backups/seerr-$(date +%F).tar.gz -C /mnt/user/appdata seerr
docker compose start seerr
```

`/app/config` holds `db/db.sqlite3` — users, request history and settings.

## Troubleshooting

**"Failed to connect" testing Radarr or Sonarr.** Container name, not IP. Confirm with `docker exec seerr wget -qO- http://radarr:7878/ping`.

**Root folder dropdown is empty.** The *arr app has no root folder set, or the API key is wrong.

**Everything shows as "Request" even though you own it.** The library scan has not run or the wrong libraries are ticked. **Settings → Media Server → Start Scan**.

**Requests approve but nothing downloads.** Seerr's job ends at handing the request to Radarr. Check Radarr's queue and history — the problem is downstream, in indexers or the download client.

**Users cannot log in after importing.** Plex users need to be actual friends on the shared server; Jellyfin users need to exist on the Jellyfin side. Seerr does not create accounts.

**Notification links point at localhost.** Application URL is unset. Step 8.

**4K requests go to the wrong place.** Seerr supports separate 4K servers — add a second Radarr entry marked as 4K rather than trying to make one profile do both.

## Where this sits in my lab

Seerr runs on the Unraid box and is the only piece of the media stack anyone else touches. It talks to Plex for authentication, since that is what the people I share with already use, and hands requests to Sonarr and Radarr.

Auto-approve is on for family, off for everyone else, with a weekly quota either way. In practice the quota has never been hit — but it is the difference between a shared library and an open-ended storage commitment, and it took thirty seconds to set.
