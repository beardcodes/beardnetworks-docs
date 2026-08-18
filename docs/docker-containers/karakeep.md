# Karakeep

Every bookmark I have ever saved has died the same way: a folder called "Read Later" with 400 links in it, half of which now 404. Karakeep fixes the second half of that problem by archiving the page content, and the first half by tagging everything automatically.

This guide covers a full Karakeep deployment — the app, its search engine, and the headless Chrome that does the crawling — plus the AI tagging setup and a reverse proxy.

[Karakeep](https://karakeep.app/) (formerly Hoarder) is a self-hosted bookmark-everything app. You throw links, notes and images at it, and it makes them findable later.

The parts that make it different from a browser's bookmark bar:

1. **Full-page archival** — stores a copy of the page, so link rot stops mattering
2. **Automatic tagging** — an LLM reads the content and applies tags, locally via Ollama or through OpenAI
3. **Full-text search** across everything it has archived, powered by Meilisearch
4. **OCR on images**, so a screenshot of a receipt is searchable by its text
5. **Browser extensions and mobile apps** for iOS and Android, plus a share target
6. **RSS feed subscriptions**, so new articles land in your inbox automatically

## How the three containers fit together

Karakeep is not one container, and understanding why saves you a lot of confusion when something breaks:

| Container | Job | Breaks if missing |
|---|---|---|
| `web` | The app, API and SQLite database | Everything |
| `chrome` | Headless Chromium that loads pages, including JS-heavy ones | Bookmarks save, but archived content and screenshots are empty |
| `meilisearch` | Search index | App works, search returns nothing |

The `web` container is the only one that needs a port published. The other two are internal.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - About 2 GB of RAM for the three containers; more if you enable AI tagging locally
    - Optional: an OpenAI API key, or an Ollama instance, for automatic tagging
    - Optional: [Traefik](traefik.md) for the reverse proxy step

## 1. Create the stack directory

```bash
sudo mkdir -p /opt/docker/karakeep && cd /opt/docker/karakeep
```

## 2. Write the Compose file

```bash
sudo nano docker-compose.yml
```

This is the upstream Compose file, unmodified:

```yaml
services:
  web:
    image: ghcr.io/karakeep-app/karakeep:${KARAKEEP_VERSION:-release}
    restart: unless-stopped
    volumes:
      - data:/data
    ports:
      - 3000:3000
    env_file:
      - .env
    environment:
      MEILI_ADDR: http://meilisearch:7700
      BROWSER_WEB_URL: http://chrome:9222
      DATA_DIR: /data
  chrome:
    image: ghcr.io/karakeep-app/karakeep-chrome:release
    restart: unless-stopped
    init: true
    command:
      - --disable-gpu
      - --disable-dev-shm-usage
      - --hide-scrollbars
      - --disable-blink-features=AutomationControlled
      - --window-size=1440,900
  meilisearch:
    image: getmeili/meilisearch:v1.41.0
    restart: unless-stopped
    env_file:
      - .env
    environment:
      MEILI_NO_ANALYTICS: "true"
    volumes:
      - meilisearch:/meili_data

volumes:
  meilisearch:
  data:
```

Two lines are worth calling out:

- `DATA_DIR: /data` — **do not change this.** If you want your bookmarks on a specific disk, change the volume mapping to `- /mnt/tank/karakeep:/data` and leave `DATA_DIR` alone. Changing the variable instead is the most common way people end up with an app that starts but cannot find its own database.
- `init: true` on `chrome` — reaps zombie Chromium processes. Without it, a long-running instance slowly accumulates defunct processes until it stops crawling.

## 3. Generate the secrets

Karakeep needs two random strings. Do not invent them by hand:

```bash
openssl rand -base64 36
openssl rand -base64 36
```

## 4. Write the .env file

```bash
sudo nano .env
```

```bash
KARAKEEP_VERSION=release
NEXTAUTH_SECRET=<first random string>
MEILI_MASTER_KEY=<second random string>
NEXTAUTH_URL=http://192.168.1.20:3000
```

What each one does:

1. `KARAKEEP_VERSION` — `release` tracks the latest stable. Pin it (`0.10.0`) if you would rather approve every upgrade yourself.
2. `NEXTAUTH_SECRET` — signs session cookies. Change it later and everyone gets logged out.
3. `MEILI_MASTER_KEY` — the API key between the app and the search engine.
4. `NEXTAUTH_URL` — **the URL you will actually type in the browser.** Get this wrong and login redirects break in a way that looks like a broken password.

Lock the file down, since it now holds secrets:

```bash
sudo chmod 600 .env
```

!!! warning "NEXTAUTH_URL must match how you reach the app"

    Using `http://localhost:3000` from another machine will fail. If you are putting it behind a proxy in step 7, this becomes `https://karakeep.example.com` — and you have to update it *and* recreate the container at that point, not before.

## 5. Start the stack

```bash
docker compose up -d
docker compose logs -f web
```

First boot runs database migrations and takes a moment. Wait for the server to report it is listening, then browse to `http://<host-ip>:3000`.

## 6. Create your account and lock signups

The first account you create is the admin. Sign up, then go to **Admin Settings** and **disable new signups** — otherwise anyone who reaches the page can create an account.

<!-- screenshot: Karakeep admin settings with signups disabled -->

You can also do it from the environment by adding `DISABLE_SIGNUPS=true` to `.env` after your account exists.

## 7. Put it behind Traefik

Add the app to your Traefik network, drop the published port, and fix `NEXTAUTH_URL`.

In `.env`:

```bash
NEXTAUTH_URL=https://karakeep.example.com
```

In `docker-compose.yml`, replace the `web` service's `ports` block with networks and labels:

```yaml
services:
  web:
    image: ghcr.io/karakeep-app/karakeep:${KARAKEEP_VERSION:-release}
    restart: unless-stopped
    volumes:
      - data:/data
    env_file:
      - .env
    environment:
      MEILI_ADDR: http://meilisearch:7700
      BROWSER_WEB_URL: http://chrome:9222
      DATA_DIR: /data
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.karakeep.rule=Host(`karakeep.example.com`)"
      - "traefik.http.routers.karakeep.entrypoints=https"
      - "traefik.http.routers.karakeep.tls.certresolver=letsencrypt"
      - "traefik.http.services.karakeep.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

The `web` container needs **both** networks: `traefik-net` to receive traffic, and the stack's `default` network to still reach `chrome` and `meilisearch`. Attaching only `traefik-net` is why people end up with a reachable app that archives nothing.

`traefik.docker.network=traefik-net` tells Traefik which of the two IPs to route to. On multi-homed containers, omitting it produces intermittent 502s that look like a flaky app.

```bash
docker compose up -d --force-recreate
```

## 8. Turn on automatic tagging

This is the feature worth having. Two routes:

**OpenAI** — cheap and accurate. Add to `.env`:

```bash
OPENAI_API_KEY=sk-...
INFERENCE_TEXT_MODEL=gpt-4o-mini
INFERENCE_IMAGE_MODEL=gpt-4o-mini
```

Tagging a few thousand bookmarks with `gpt-4o-mini` costs a few dollars in total.

**Ollama** — free, private, slower. Point Karakeep at an existing Ollama server:

```bash
OLLAMA_BASE_URL=http://192.168.1.30:11434
INFERENCE_TEXT_MODEL=llama3.1
INFERENCE_IMAGE_MODEL=llava
INFERENCE_CONTEXT_LENGTH=4096
```

Recreate the container after either change. New bookmarks get tagged on arrival; existing ones can be re-processed from the admin panel.

!!! note "Ollama needs headroom"

    `INFERENCE_CONTEXT_LENGTH` defaults low. An article longer than the context window gets truncated before the model sees it, and the tags come back generic. 4096 is a reasonable floor; raise it if you have the VRAM.

## 9. Install the browser extension

Karakeep's extensions for [Chrome](https://chromewebstore.google.com/detail/karakeep/kgcjekpmcjjogibpjebkhaanilehneje) and [Firefox](https://addons.mozilla.org/en-US/firefox/addon/karakeep/) ask for your server address and an API key. Generate the key under **Settings → API Keys**.

The mobile apps use the same key, and register as a system share target — which is the point at which you actually start using the thing.

## Updating

```bash
cd /opt/docker/karakeep
docker compose pull
docker compose up -d
```

If you pinned `KARAKEEP_VERSION`, bump it in `.env` first. Check the [release notes](https://github.com/karakeep-app/karakeep/releases) before jumping a minor version — this project moves fast and occasionally changes environment variable names.

## Backup

Everything that matters is in the `data` volume: the SQLite database and the archived page content. The Meilisearch volume is a rebuildable index, so skip it.

```bash
cd /opt/docker/karakeep
docker compose stop web
docker run --rm \
  -v karakeep_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/karakeep-$(date +%F).tar.gz -C /data .
docker compose start web
```

Stopping `web` first matters — SQLite does not enjoy being copied mid-write. Check your real volume name with `docker volume ls` if the stack directory is not called `karakeep`.

## Troubleshooting

**Bookmarks save but stay blank, no screenshot.** The `chrome` container. Check `docker compose logs chrome`, and confirm `BROWSER_WEB_URL` is `http://chrome:9222`.

**Search finds nothing.** Meilisearch is unreachable or the key does not match. `docker compose logs meilisearch`, and confirm `MEILI_MASTER_KEY` is identical for both services — they read the same `.env`, so this usually means a stale container.

**Login redirects to the wrong host, or a redirect loop.** `NEXTAUTH_URL` does not match the address in your browser bar. Fix it and recreate the container; a restart is not enough.

**Crawling fails on some sites only.** Those sites are blocking headless Chrome. `--disable-blink-features=AutomationControlled` in the Compose file helps, and is already there. Some sites will simply refuse.

**Container restarts on a low-RAM host.** Chromium is the memory hog. Give the VM 4 GB, or set `CRAWLER_NUM_WORKERS=1` to stop it opening several pages at once.

## Where this sits in my lab

Karakeep runs on the Docker VM on my first Proxmox node, alongside Home Assistant and Change Detection. It sits behind Traefik with an AdGuard rewrite so `karakeep.home.lan` resolves internally, and it is not exposed to the internet — the mobile apps reach it over WireGuard.

It replaced a Firefox bookmark folder, a Notion database and about six "I'll read this later" browser tabs that had been open since last year.
