# Homepage

Homepage is the other end of the dashboard spectrum from [Glance](glance.md). Where Glance is feeds and widgets, Homepage is about your services — and specifically about showing live data *from* them. Not just a link to Sonarr, but Sonarr's queue length. Not just a link to Proxmox, but its CPU and memory.

It reads that data through service widgets, of which there are over a hundred.

[Homepage](https://gethomepage.dev/) is a highly customisable application dashboard with service integrations, configured in YAML.

1. **100+ service widgets** — the *arr stack, Proxmox, Unraid, Plex, Uptime Kuma, AdGuard, and most of what is in this documentation
2. **Docker integration** — discovers containers and shows their status, optionally via labels
3. **Kubernetes integration** for cluster resources
4. **Information widgets** — system resources, weather, search, bookmarks
5. **YAML config**, so the whole dashboard is version-controllable
6. **Fast static rendering**, with per-widget refresh

## Homepage or Glance?

Genuinely different tools that look superficially similar:

| | Homepage | Glance |
|---|---|---|
| Core idea | Your services, with live data from their APIs | Feeds, widgets and information |
| Service widgets | 100+, deep API integrations | Basic up/down monitoring |
| RSS, Reddit, HN, YouTube | Limited | Excellent |
| Docker discovery | Yes, with labels | Yes, basic |
| Config | Several YAML files | One YAML file |

Homepage answers "what is my lab doing right now?". Glance answers "what should I read, and is everything up?". Running both is not unreasonable — they are different pages for different moments.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - API keys for the services you want live data from

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/homepage/config && cd /opt/docker/homepage
sudo nano docker-compose.yml
```

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - 3001:3000
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      HOMEPAGE_ALLOWED_HOSTS: home.example.com,192.168.1.30:3001
```

!!! danger "HOMEPAGE_ALLOWED_HOSTS is required and will break your install if wrong"

    Since version 1.0 this variable is **mandatory**. If the `Host` header of a request is not in the list, Homepage returns an error instead of the dashboard.

    The symptom is a blank page or a host-validation error immediately after a working install suddenly stops working — usually right after you put it behind a reverse proxy and the hostname changes.

    Include **every** hostname and port you will use to reach it: the internal IP with port, the proxied hostname, and `localhost` if you test locally. Comma-separated, no spaces.

Note port `3001:3000` — [Dockhand](dockhand.md), [Grafana](grafana.md) and [Karakeep](karakeep.md) are all competing for 3000.

The Docker socket is optional and read-only. Upstream notes that mounting it directly is not their recommended integration method and requires running as root or adding the user to the docker group — a socket proxy is the safer pattern if that bothers you.

```bash
docker compose up -d
```

Homepage writes default config files into `./config` on first start. Browse to `http://<host-ip>:3001`.

## 2. The config files

Homepage splits configuration across several files in `config/`:

| File | Contains |
|---|---|
| `services.yaml` | Your services, grouped, with widgets |
| `widgets.yaml` | The header — resources, search, weather |
| `settings.yaml` | Layout, theme, title |
| `bookmarks.yaml` | Link groups |
| `docker.yaml` | Docker host connections |

## 3. Add services with live widgets

`services.yaml` is where the value is. This is not just links — each `widget` block pulls live data from that service's API:

```yaml
- Media:
    - Seerr:
        icon: jellyseerr.png
        href: https://requests.example.com
        description: Request management
        widget:
          type: jellyseerr
          url: http://192.168.1.50:5055
          key: {{HOMEPAGE_VAR_SEERR_KEY}}

    - Sonarr:
        icon: sonarr.png
        href: http://192.168.1.50:8989
        description: TV
        widget:
          type: sonarr
          url: http://192.168.1.50:8989
          key: {{HOMEPAGE_VAR_SONARR_KEY}}

    - Radarr:
        icon: radarr.png
        href: http://192.168.1.50:7878
        description: Films
        widget:
          type: radarr
          url: http://192.168.1.50:7878
          key: {{HOMEPAGE_VAR_RADARR_KEY}}

- Infrastructure:
    - Proxmox:
        icon: proxmox.png
        href: https://192.168.1.10:8006
        widget:
          type: proxmox
          url: https://192.168.1.10:8006
          username: api@pam!homepage
          password: {{HOMEPAGE_VAR_PROXMOX_TOKEN}}

    - AdGuard:
        icon: adguard-home.png
        href: http://192.168.1.5
        widget:
          type: adguard
          url: http://192.168.1.5
          username: admin
          password: {{HOMEPAGE_VAR_ADGUARD_PASS}}

    - Uptime Kuma:
        icon: uptime-kuma.png
        href: http://192.168.1.30:3001
        widget:
          type: uptimekuma
          url: http://192.168.1.30:3001
          slug: homelab
```

Sonarr's widget shows the queue and how many episodes are missing. Proxmox shows VM counts and resource use. AdGuard shows queries blocked today. That is the difference from a page of bookmarks.

<!-- screenshot: Homepage dashboard with live service widgets showing queue counts -->

## 4. Keep API keys out of the config

Those `{{HOMEPAGE_VAR_*}}` placeholders are substituted from the environment, which means `services.yaml` stays safe to commit to Git.

In `.env`:

```bash
HOMEPAGE_VAR_SONARR_KEY=abc123...
HOMEPAGE_VAR_RADARR_KEY=def456...
HOMEPAGE_VAR_SEERR_KEY=ghi789...
HOMEPAGE_VAR_ADGUARD_PASS=...
HOMEPAGE_VAR_PROXMOX_TOKEN=...
```

And in Compose:

```yaml
    env_file:
      - .env
```

```bash
chmod 600 .env
```

Without this, every API key in your lab ends up in a file you will eventually push to a public repository.

## 5. The header widgets

`widgets.yaml` controls the strip across the top:

```yaml
- resources:
    cpu: true
    memory: true
    disk: /

- search:
    provider: duckduckgo
    target: _blank

- openmeteo:
    label: London
    latitude: 51.5074
    longitude: -0.1278
    units: metric
    cache: 5
```

`openmeteo` needs no API key, unlike the older weather widgets.

## 6. Docker labels instead of config

Services can define their own dashboard entry, which keeps the definition next to the container:

```yaml
    labels:
      - "homepage.group=Media"
      - "homepage.name=Sonarr"
      - "homepage.icon=sonarr.png"
      - "homepage.href=http://192.168.1.50:8989"
      - "homepage.widget.type=sonarr"
      - "homepage.widget.url=http://sonarr:8989"
      - "homepage.widget.key=${SONARR_KEY}"
```

Requires the Docker socket and a `docker.yaml` entry. Nice in principle; in practice a single `services.yaml` is easier to reason about when things go wrong, and it works for services that are not containers.

## 7. Put it behind Traefik

```yaml
    environment:
      HOMEPAGE_ALLOWED_HOSTS: home.example.com
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.homepage.rule=Host(`home.example.com`)"
      - "traefik.http.routers.homepage.entrypoints=https"
      - "traefik.http.routers.homepage.tls.certresolver=letsencrypt"
      - "traefik.http.services.homepage.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

**Update `HOMEPAGE_ALLOWED_HOSTS` at the same time**, or you will proxy successfully to an error page. This is the number one Homepage support question.

Homepage has no authentication. It displays live data about your infrastructure and links to every admin panel you own — keep it on the LAN, or put [Tinyauth](tinyauth.md) in front.

## Updating

```bash
cd /opt/docker/homepage
docker compose pull
docker compose up -d
```

Homepage has had breaking config changes, `HOMEPAGE_ALLOWED_HOSTS` being the most disruptive. Skim the release notes on major versions.

## Backup

The `config` directory is the entire application.

```bash
sudo tar czf /mnt/user/backups/homepage-$(date +%F).tar.gz \
  -C /opt/docker/homepage config .env
```

Better: keep `config/` in Git. Because the API keys are in `.env`, the YAML is safe to commit.

## Troubleshooting

**Blank page or a host error.** `HOMEPAGE_ALLOWED_HOSTS`. Include the exact host and port from your browser's address bar.

**A widget shows "API Error".** Wrong key, or Homepage cannot reach the service. Test from inside: `docker exec homepage wget -qO- http://192.168.1.50:8989/api/v3/system/status?apikey=xxx`.

**Widget works for one app, not another.** Each widget expects a specific API version. Check the [widget docs](https://gethomepage.dev/widgets/) for that service — some need a URL with no trailing slash, or a specific path.

**Icons not loading.** Use a name from the [dashboard-icons](https://github.com/homarr-labs/dashboard-icons) set, e.g. `sonarr.png`, or a full URL.

**Docker widget empty.** Socket not mounted, or a permissions problem — Homepage runs as non-root when PUID/PGID are set.

**Config changes not appearing.** Homepage watches the files, but a YAML syntax error causes it to keep the last good config. Check the logs.

**Proxmox widget fails with 401.** Use an API token, not a password, and the username must be the full `user@realm!tokenid` form.

## Where this sits in my lab

Homepage runs on `proxmox2` and is the page I open when I want to know what the lab is doing — queue lengths, disk use, VM states, blocked queries.

[Glance](glance.md) is the browser home page for reading; Homepage is the operations view. The overlap is small enough that keeping both has not felt redundant, though I would not argue hard against someone who picked one.

`config/` lives in Gitea with the keys in `.env`, so the dashboard is reproducible from the repo.
