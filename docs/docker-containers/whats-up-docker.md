# What's Up Docker

[Diun](diun.md) tells you when a container image has been updated. What's Up Docker does the same job, adds a web UI, understands semantic versioning, and — if you let it — applies the update itself.

Whether that last part is a feature or a hazard is the interesting question, and this page takes a position on it.

!!! note "The project moved"

    What's Up Docker used to live at `fmartinou/whats-up-docker`. It is now **`getwud/wud`**, with images published as `getwud/wud` on Docker Hub and `ghcr.io/getwud/wud` on GitHub Container Registry. Older guides pointing at the previous repository are out of date.

[What's Up Docker](https://github.com/getwud/wud) keeps your containers up to date.

1. **Watches running containers** and checks their registries for newer tags
2. **Semantic version aware** — distinguishes a patch from a major, so you can act differently on each
3. **Web UI** listing every container and what update is available
4. **Triggers** — notify via ntfy, Discord, Slack, email, or act via webhook, Kafka, MQTT
5. **Auto-update trigger** that recreates the container on a new image
6. **Registry support** for Docker Hub, GHCR, ECR, GCR, ACR, Quay and private registries

## WUD or Diun?

You already run [Diun](diun.md), so the honest comparison:

| | What's Up Docker | Diun |
|---|---|---|
| Web UI | Yes | No, notifications only |
| Semver awareness | Yes — knows major vs minor vs patch | Tag-based |
| Auto-update | Yes | No, by design |
| API | Yes | No |
| Resource use | Slightly higher | Very small |
| Configuration | Environment variables or labels | A YAML file |

Diun is a notifier and nothing else, which is a defensible design. WUD does more, and the UI genuinely helps when you have thirty containers and want to see the whole update picture at once.

Running both is redundant. Pick based on whether you want a dashboard.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - Port 3000 free, or a remap

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/wud && cd /opt/docker/wud
sudo nano docker-compose.yml
```

Upstream's quickstart:

```yaml
services:
  whatsupdocker:
    image: getwud/wud
    container_name: wud
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    ports:
      - 3000:3000
```

A more considered version — read-only socket, restart policy, and a port that does not fight [Grafana](grafana.md), [Dockhand](dockhand.md) and [Homepage](homepage.md):

```yaml
services:
  wud:
    image: getwud/wud
    container_name: wud
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - wud_store:/store
    ports:
      - 3003:3000
    environment:
      - TZ=Europe/London
      - WUD_WATCHER_LOCAL_CRON=0 6 * * *

volumes:
  wud_store:
```

**Mount the socket read-only** unless you intend to use auto-update. WUD only needs to *read* container metadata to report updates; write access is what enables it to recreate containers. Start read-only, and make granting write access a deliberate decision rather than a default.

```bash
docker compose up -d
```

Browse to `http://<host-ip>:3003`.

## 2. What the UI shows

Every running container, its current tag, and whether a newer image exists — with the semver relationship marked, so a `1.2.3 → 1.2.4` patch is visually distinct from `1.2.3 → 2.0.0`.

That distinction is the reason to prefer WUD over a plain notifier: a patch on Uptime Kuma and a major version on your database are not the same event, and a notification that treats them identically trains you to ignore both.

<!-- screenshot: WUD container list showing available updates with semver labels -->

## 3. Add a notification trigger

Triggers are configured as environment variables, in the pattern `WUD_TRIGGER_<TYPE>_<NAME>_<SETTING>`:

```yaml
    environment:
      - TZ=Europe/London
      - WUD_TRIGGER_NTFY_HOMELAB_URL=https://ntfy.example.com
      - WUD_TRIGGER_NTFY_HOMELAB_TOPIC=homelab
      - WUD_TRIGGER_NTFY_HOMELAB_PRIORITY=3
```

`HOMELAB` is a name you choose; use it consistently across the three lines. Discord, Slack, SMTP, MQTT and generic webhooks follow the same shape — the [trigger docs](https://getwud.github.io/wud/#/configuration/triggers/) list the settings for each.

## 4. Control what gets watched

By default WUD watches everything. Per-container labels refine that:

```yaml
    labels:
      - "wud.watch=true"
      - "wud.tag.include=^\\d+\\.\\d+\\.\\d+$$"
      - "wud.link.template=https://github.com/org/repo/releases/tag/$${major}.$${minor}.$${patch}"
```

`wud.tag.include` is the useful one. A container pinned to `:15` should not report `:latest` as an update, and a regex restricting it to semver tags stops the noise. Note the doubled `$$` for Compose.

To exclude a container entirely, `wud.watch=false`.

## 5. Auto-update — think before enabling

WUD can recreate a container when a new image appears:

```yaml
      - WUD_TRIGGER_DOCKER_UPDATE_PRUNE=true
```

with `wud.watch.digest=true` and the auto-update trigger on containers you opt in.

!!! danger "Do not auto-update stateful services"

    Unattended updates are fine for a static, stateless container. They are a bad idea for anything with a database, because a major version can run an irreversible schema migration while you are asleep, and the rollback is "restore last night's backup".

    Never auto-update: [Vaultwarden](vaultwarden.md), [Firefly III](firefly-iii.md), [Immich](https://immich.app/), [Mealie](mealie.md), the *arr apps, or anything else holding data you cannot rebuild.

    Reasonable candidates: [IT-Tools](it-tools.md), [BentoPDF](bentopdf.md), [Excalidraw](excalidraw.md) — static things with no persistent state, where the worst case is a rollback of the tag.

    Even then, take the backup first. The safe default is notify-only, and doing the update yourself when you have read the release notes.

## 6. Put it behind Traefik

```yaml
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wud.rule=Host(`wud.home.lan`)"
      - "traefik.http.routers.wud.entrypoints=https"
      - "traefik.http.routers.wud.tls.certresolver=letsencrypt"
      - "traefik.http.services.wud.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

WUD has optional basic auth, but it exposes an inventory of everything you run and their versions — which is a useful reconnaissance document. Keep it on the LAN or behind [Tinyauth](tinyauth.md).

## Updating

```bash
cd /opt/docker/wud
docker compose pull
docker compose up -d
```

There is a pleasing circularity in WUD telling you that WUD needs updating.

## Backup

Almost nothing to keep — the store holds cached state that rebuilds on the next scan. The configuration is your Compose file, so keep that in Git.

```bash
sudo cp /opt/docker/wud/docker-compose.yml /mnt/user/backups/
```

## Troubleshooting

**No containers listed.** Socket not mounted, or permissions. `docker exec wud ls -l /var/run/docker.sock`.

**Everything reports an update available, constantly.** Containers on `:latest`. WUD compares digests, and `:latest` changes often. Pin real version tags — this is a signal your tagging is too loose, not a WUD bug.

**Registry rate limit errors.** Anonymous Docker Hub pulls are limited. Add credentials via `WUD_REGISTRY_HUB_LOGIN` and `WUD_REGISTRY_HUB_TOKEN`, or lower the scan frequency.

**Private registry images not checked.** Configure that registry explicitly; WUD cannot guess credentials.

**Notifications fire repeatedly for the same update.** Expected until the container is actually updated. Adjust the cron, or update the thing.

**Trigger does not fire at all.** The name segment must match across all its variables. `WUD_TRIGGER_NTFY_HOMELAB_URL` and `WUD_TRIGGER_NTFY_HOME_TOPIC` define two different half-configured triggers.

## Where this sits in my lab

WUD runs on the Docker VM on my first Proxmox node with the socket mounted read-only and **auto-update off everywhere**. It notifies to ntfy at low priority — update notifications are not an emergency, and mixing them with [Uptime Kuma](uptime-kuma.md)'s "something is down" alerts would devalue both.

The semver awareness is the reason it earns its place over a plain notifier. Patch updates I apply in a batch when convenient; major versions get the release notes read first, which is exactly the distinction I want a tool to make for me.
