# Glance

A homelab accumulates URLs. Thirty services, each on a different port, none of them memorable. A dashboard fixes that — and Glance goes further by pulling in the things you would otherwise open six tabs to check: RSS feeds, server stats, weather, release notes for the software you run.

It is configured entirely in one YAML file, which means it belongs in your Git repo rather than in a database.

[Glance](https://github.com/glanceapp/glance) is a self-hosted dashboard that puts all your feeds in one place.

1. **Widgets for everything** — RSS, Reddit, Hacker News, YouTube channels, weather, calendar, markets, Docker containers
2. **Monitor widget** — service up/down status with response times, so it doubles as a light status page
3. **Releases widget** — tracks new versions of the GitHub projects you run, which is [Diun](diun.md) for things that are not containers
4. **Fast** — a single Go binary, minimal memory, pages load instantly
5. **Themeable**, with a config-file theme rather than a settings screen
6. **YAML configuration**, so the whole dashboard is version-controllable

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/glance && cd /opt/docker/glance
sudo nano docker-compose.yml
```

```yaml
services:
  glance:
    container_name: glance
    image: glanceapp/glance
    restart: unless-stopped
    volumes:
      - ./config:/app/config
    ports:
      - 8080:8080
```

If you want the Docker containers widget, add the socket read-only:

```yaml
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Port 8080 collides with a great many things — [qBittorrent](qbittorrent.md) and cAdvisor among them. Remap the host side if needed.

## 2. Get the starter config

Glance will not start without a config file:

```bash
mkdir config && wget -O config/glance.yml \
  https://raw.githubusercontent.com/glanceapp/glance/refs/heads/main/docs/glance.yml
```

```bash
docker compose up -d
docker compose logs -f
```

Browse to `http://<host-ip>:8080`. The starter config is a reasonable demonstration; the point is to replace it.

## 3. Understand the structure

Glance's config is pages, columns, widgets:

```yaml
pages:
  - name: Home
    columns:
      - size: small
        widgets:
          - type: calendar

      - size: full
        widgets:
          - type: rss
            title: News
            feeds:
              - url: https://news.ycombinator.com/rss

      - size: small
        widgets:
          - type: weather
            location: London, United Kingdom
```

Column sizes must add up sensibly: either `full` alone, or one `full` with `small` columns either side. Three `full` columns is not a layout Glance supports.

Config changes are picked up on restart:

```bash
docker compose restart glance
```

## 4. A homelab dashboard worth having

The generic config is a demo. This is closer to useful — service links, live status, and update tracking in one screen:

```yaml
pages:
  - name: Homelab
    columns:
      - size: small
        widgets:
          - type: clock
            timezone: Europe/London

          - type: monitor
            cache: 1m
            title: Infrastructure
            sites:
              - title: Proxmox
                url: https://192.168.1.10:8006
                icon: si:proxmox
                allow-insecure: true
              - title: Unraid
                url: http://192.168.1.50
                icon: si:unraid
              - title: AdGuard
                url: http://192.168.1.5
                icon: si:adguard

      - size: full
        widgets:
          - type: monitor
            cache: 1m
            title: Services
            sites:
              - title: Seerr
                url: http://192.168.1.50:5055
                icon: si:jellyfin
              - title: Sonarr
                url: http://192.168.1.50:8989
                icon: si:sonarr
              - title: Radarr
                url: http://192.168.1.50:7878
                icon: si:radarr
              - title: Grafana
                url: http://192.168.1.30:3000
                icon: si:grafana

          - type: releases
            cache: 1d
            repositories:
              - glanceapp/glance
              - go-gitea/gitea
              - louislam/uptime-kuma
              - immich-app/immich
              - mealie-recipes/mealie

      - size: small
        widgets:
          - type: weather
            location: London, United Kingdom

          - type: docker-containers
            hide-by-default: false
```

The **releases** widget is the underrated one. [Diun](diun.md) tells you when a container image changes; this tells you when the *project* cuts a release, which is when the changelog you actually want to read appears.

<!-- screenshot: Glance homelab dashboard with monitor, releases and weather widgets -->

## 5. Icons

Glance supports a few icon sources, and the prefix selects which:

| Prefix | Source | Example |
|---|---|---|
| `si:` | Simple Icons | `si:docker` |
| `di:` | Dashboard Icons — the self-hosted set | `di:sonarr` |
| `sh:` | Selfh.st icons | `sh:radarr` |
| none | A URL to your own image | `/assets/thing.png` |

`di:` has the best coverage for homelab software specifically. Simple Icons covers brands but not every *arr app.

## 6. Multiple pages

One dashboard becomes crowded fast. Split it:

```yaml
pages:
  - name: Home
    columns: [...]

  - name: Media
    columns: [...]

  - name: Infra
    columns: [...]
```

Pages appear as tabs across the top. A common split is Home for daily use, Media for the *arr stack, and Infra for hypervisors and monitoring.

## 7. Put it behind Traefik

```yaml
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.glance.rule=Host(`home.example.com`)"
      - "traefik.http.routers.glance.entrypoints=https"
      - "traefik.http.routers.glance.tls.certresolver=letsencrypt"
      - "traefik.http.services.glance.loadbalancer.server.port=8080"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Glance has **no authentication of its own.** It is a page full of links to your internal infrastructure, so either keep it on the LAN, or put [Tinyauth](tinyauth.md) in front of it.

Setting it as your browser's home page is the entire point, so make the hostname short.

## Updating

```bash
cd /opt/docker/glance
docker compose pull
docker compose up -d
```

Glance moves quickly and has had breaking config changes — the project ships a [v0.7.0 upgrade guide](https://github.com/glanceapp/glance/blob/main/docs/v0.7.0-upgrade.md) for one of them. If the container starts failing after a pull, read the release notes before assuming your config is corrupt.

## Backup

The config file is the whole application state:

```bash
sudo cp /opt/docker/glance/config/glance.yml /mnt/user/backups/
```

Better: keep `glance.yml` in a Git repo. It is a text file that fully defines your dashboard, which is exactly what version control is for.

## Troubleshooting

**Container will not start.** Almost always a YAML error. `docker compose logs glance` names the line. Indentation, usually.

**A widget shows an error or stays empty.** That widget's source is unreachable or rate-limited. Reddit and GitHub both rate-limit unauthenticated requests; raise the `cache` duration.

**Monitor widget shows everything down.** Glance checks from inside its container. `docker exec glance wget -qO- http://192.168.1.50:8989` to confirm what it can actually see. Use IPs or container names, not hostnames that only resolve on your workstation.

**Monitor shows a self-signed HTTPS service as down.** Add `allow-insecure: true` to that site.

**Docker containers widget is empty.** The socket is not mounted, or has no read permission.

**Layout looks wrong.** Column sizes. One `full` per page, `small` either side.

**Changes not appearing.** Restart the container — config is read at startup.

## Where this sits in my lab

Glance runs on `proxmox2` and is the browser home page on every machine in the house. Three pages: Home with clock, weather and feeds; Media with the *arr stack and Seerr; Infra with the hypervisors, monitoring and releases.

It overlaps with [Uptime Kuma](uptime-kuma.md) on the monitor widget, and that is fine — the widget is a glance, Kuma is the alerting. One tells me something is red while I am already looking; the other wakes me up.

`glance.yml` lives in Gitea, which means the dashboard survives the container, the volume, and the host.
