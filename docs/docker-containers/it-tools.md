# IT-Tools

Everyone has a bookmark folder of single-purpose websites: a JSON formatter, a base64 decoder, a JWT inspector, a UUID generator, a crontab explainer. They are all covered in ads, several are abandoned, and at least one of them is quietly logging what you paste into it.

IT-Tools is all of them, in one container, on your own network.

[IT-Tools](https://it-tools.tech/) is a collection of handy tools for developers, with a genuinely good interface.

1. **80+ tools** — converters, generators, encoders, parsers, network calculators
2. **Runs client-side**, so what you paste stays in your browser
3. **Searchable** with a keyboard-first palette
4. **Favourites** pinned to the top, stored per browser
5. **Dark mode**, and a layout that works on a phone
6. **One container, no state** — nothing to configure, nothing to back up

## What is actually in it

Enough to replace most of that bookmark folder:

| Category | Examples |
|---|---|
| Crypto | Hashes, bcrypt, HMAC, RSA key pairs, password generator |
| Converters | Base64, JSON ↔ YAML ↔ TOML, timestamps, number bases, colours |
| Web | JWT parser, URL encode/decode, MIME types, Basic auth generator |
| Network | Subnet calculator, IPv4/IPv6 tools, MAC lookup |
| Text | Diff, case conversion, regex tester, Lorem ipsum |
| Dev | Crontab generator, chmod calculator, Docker run ↔ Compose converter, SQL formatter |
| Media | QR codes, SVG placeholders |

The **Docker run to Compose converter** is worth knowing about on its own — most upstream projects document `docker run`, and half the work in writing the guides in this documentation is translating those into Compose files.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)

## 1. The Compose file

Upstream documents `docker run`:

```bash
docker run -d --name it-tools --restart unless-stopped -p 8080:80 ghcr.io/corentinth/it-tools:latest
```

As Compose:

```bash
sudo mkdir -p /opt/docker/it-tools && cd /opt/docker/it-tools
sudo nano docker-compose.yml
```

```yaml
services:
  it-tools:
    image: ghcr.io/corentinth/it-tools:latest
    container_name: it-tools
    restart: unless-stopped
    ports:
      - 8084:80
```

That is the entire deployment. **No volumes, no environment variables, no database** — it is a static site served by nginx, and every tool runs in your browser.

Port 8080 is contested by [Glance](glance.md), [qBittorrent](qbittorrent.md) and cAdvisor, so this uses 8084 on the host.

The image is also on Docker Hub as `corentinth/it-tools:latest` if you prefer.

```bash
docker compose up -d
```

Browse to `http://<host-ip>:8084`.

## 2. Verify it runs locally

The same check as [BentoPDF](bentopdf.md), and worth doing once: open developer tools, go to the **Network** tab, and hash a string or decode a JWT.

No requests should leave the page. That is the entire reason to self-host this rather than use the public instance — a JWT pasted into a random website is a credential handed to a stranger, and people do it constantly.

Disconnect the machine from the network and the tools keep working.

## 3. Pin your favourites

Click the star on any tool and it moves to a favourites section at the top.

Stored in browser local storage, not on the server — so favourites are per-browser, not per-user, and clearing site data resets them. With no accounts and no database, that is the trade.

<!-- screenshot: IT-Tools home with favourites pinned and the tool search open -->

## 4. Put it behind Traefik

```yaml
services:
  it-tools:
    image: ghcr.io/corentinth/it-tools:latest
    container_name: it-tools
    restart: unless-stopped
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.it-tools.rule=Host(`tools.home.lan`)"
      - "traefik.http.routers.it-tools.entrypoints=https"
      - "traefik.http.routers.it-tools.tls.certresolver=letsencrypt"
      - "traefik.http.services.it-tools.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Serve it over HTTPS — some browser crypto APIs require a secure context, and a few tools will misbehave on plain HTTP.

Nothing to protect with [Tinyauth](tinyauth.md): no accounts, no stored data, no state. A login here would be friction with no security benefit. A short hostname matters more, since the whole point is reaching it faster than searching for an online equivalent.

## Updating

```bash
cd /opt/docker/it-tools
docker compose pull
docker compose up -d
```

About as safe as updating gets — no database, no migrations, no config. New releases mostly add tools.

Hard-refresh the browser afterwards (Ctrl+Shift+R); the static assets cache aggressively.

## Backup

Nothing to back up. Keep the six-line Compose file in Git and the service is fully reproducible.

## Troubleshooting

**Port 8080 already allocated.** Something else has it. `ss -tlnp | grep 8080`, then remap the host side as above.

**A crypto tool fails or does nothing.** Serve over HTTPS. `crypto.subtle` is unavailable in insecure contexts.

**Favourites disappeared.** Local storage was cleared, or you are in a different browser or a private window. Expected behaviour.

**New tools missing after an update.** Cached assets. Hard refresh.

**Blank page behind the reverse proxy.** Check you are proxying to port 80, the container's port, not the host mapping.

## Where this sits in my lab

IT-Tools runs on the Docker VM on my first Proxmox node at `tools.home.lan`, LAN-only, alongside [BentoPDF](bentopdf.md) — the two "stop using a random website for this" services.

Both exist for the same reason. The online versions are free because the input is the product, and the input is frequently a token, a hash, or a config file you should not be pasting into someone else's server. Two containers, no state between them, and the habit goes away.
