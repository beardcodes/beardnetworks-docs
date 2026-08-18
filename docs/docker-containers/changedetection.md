# Change Detection

Some websites still have no RSS feed, no API and no notification option. A restock page, a council planning portal, a firmware download page, the changelog for a piece of software you depend on. Change Detection watches those pages for you and tells you the moment the text moves.

This guide covers a standard deployment, adding a real Chrome browser for JavaScript-heavy sites, wiring up notifications, and the visual selector that makes the whole thing usable.

[changedetection.io](https://changedetection.io/) polls a URL on a schedule, diffs the result against the last version, and notifies you when something you care about changed.

Why it beats a cron job and `diff`:

1. **Visual selector** — click the element on a rendered screenshot of the page, and it watches only that
2. **Restock and price mode** — purpose-built for "tell me when this is back in stock or under £X"
3. **Notifications to 80+ services** via Apprise — ntfy, Discord, Telegram, email, webhooks
4. **Diff view** with word-level highlighting and full history
5. **Filters** — CSS selectors, XPath, JSON path, and ignore rules for the cookie banner that changes every load
6. **Real browser fetching** for sites that render everything client-side

## Two ways to fetch a page

This is the decision that determines whether the tool works on your target site:

| | Plain requests (default) | Real browser |
|---|---|---|
| Extra container | None | `sockpuppetbrowser`, ~700 MB RAM |
| Handles JavaScript | No | Yes |
| Screenshots | No | Yes |
| Visual selector | No | Yes |
| Speed | Fast | Slower |

Start with plain requests. Add the browser when you hit a site that returns an empty page — which, for anything built in the last five years, is most of them.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - 512 MB RAM for the base app, ~1.5 GB if you add the browser
    - Optional: an ntfy or Discord endpoint for notifications
    - Optional: [Traefik](traefik.md) for the reverse proxy step

## 1. Create the stack directory

```bash
sudo mkdir -p /opt/docker/changedetection && cd /opt/docker/changedetection
```

## 2. Write the Compose file

```bash
sudo nano docker-compose.yml
```

Start with the minimal version:

```yaml
services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io
    container_name: changedetection
    hostname: changedetection
    restart: unless-stopped
    volumes:
      - changedetection-data:/datastore
    environment:
      - BASE_URL=http://192.168.1.20:5000
    ports:
      - 127.0.0.1:5000:5000

volumes:
  changedetection-data:
```

Two details:

- `127.0.0.1:5000:5000` — upstream binds to localhost only, deliberately. The app has **no authentication until you set a password**, so this default stops you accidentally publishing an open instance. To reach it from another machine on your LAN, change it to `5000:5000` — and then immediately do step 4.
- `BASE_URL` — gets embedded in notifications so the "view diff" link in your phone alert actually resolves. Without it you get relative links that go nowhere.

## 3. Start it

```bash
docker compose up -d
docker compose logs -f
```

If you left the localhost bind in place, tunnel in from your workstation:

```bash
ssh -L 5000:localhost:5000 user@192.168.1.20
```

Then open `http://localhost:5000`.

## 4. Set a password immediately

**Settings → General → Password**. There is no user account, just a single password that gates the UI.

!!! danger "No password means no protection"

    Change Detection ships unauthenticated. Anyone who reaches the port can read every page you watch, see your notification URLs — which frequently contain API tokens — and add watches that make your server fetch arbitrary URLs. Set the password before you change the port binding, not after.

## 5. Add your first watch

Paste a URL into the box and hit **Watch**. Defaults to every 3 hours; change it per-watch or globally under **Settings → General**.

Once it has fetched at least twice, the **Diff** view shows what moved. Be polite with the interval — 5 minutes on someone's small site is rude, and will get your IP blocked.

<!-- screenshot: changedetection.io watch list with a diff highlighted -->

## 6. Add a real browser for JavaScript sites

If a watch returns an empty page or the same content forever, the site renders client-side. Add the browser container:

```yaml
services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io
    container_name: changedetection
    hostname: changedetection
    restart: unless-stopped
    volumes:
      - changedetection-data:/datastore
    environment:
      - BASE_URL=http://192.168.1.20:5000
      - PLAYWRIGHT_DRIVER_URL=ws://sockpuppetbrowser:3000
    ports:
      - 5000:5000
    depends_on:
      - sockpuppetbrowser

  sockpuppetbrowser:
    image: dgtlmoon/sockpuppetbrowser:latest
    hostname: sockpuppetbrowser
    container_name: sockpuppetbrowser
    restart: unless-stopped
    cap_add:
      - SYS_ADMIN
    environment:
      - SCREEN_WIDTH=1920
      - SCREEN_HEIGHT=1024
      - SCREEN_DEPTH=16
      - MAX_CONCURRENT_CHROME_PROCESSES=10

volumes:
  changedetection-data:
```

```bash
docker compose up -d
```

Existing watches do not switch automatically. Edit a watch and set **Fetch Method** to *Playwright Chromium*, or change the default under **Settings → Fetching**.

!!! warning "SYS_ADMIN is a broad capability"

    Chromium's sandbox wants it. Upstream notes it may be more than strictly needed on your platform. If that bothers you, keep this container off any network segment you care about — it is the one thing in the stack that deliberately loads untrusted web content.

Once the browser is running, the **Visual Selector** appears in each watch's edit screen. Click the element you want to track and it writes the CSS selector for you. This is far more reliable than watching a whole page, because it ignores the ad rotation and the "12 people are viewing this" counter.

## 7. Notifications

**Settings → Notifications**, using [Apprise](https://github.com/caronc/apprise/wiki) URL syntax. A few that work well:

```
ntfys://ntfy.example.com/homelab
discord://webhook_id/webhook_token
tgram://bottoken/ChatID
mailtos://user:pass@smtp.example.com?to=me@example.com
```

The body supports tokens — `{{watch_url}}`, `{{diff}}`, `{{diff_url}}`, `{{watch_title}}`:

```
{{watch_title}} changed

{{diff}}

{{diff_url}}
```

Use **Send test notification** before trusting it. A silent notification failure is indistinguishable from a page that never changes.

## 8. Put it behind Traefik

```yaml
services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io
    container_name: changedetection
    hostname: changedetection
    restart: unless-stopped
    volumes:
      - changedetection-data:/datastore
    environment:
      - BASE_URL=https://changes.example.com
      - USE_X_SETTINGS=1
      - PLAYWRIGHT_DRIVER_URL=ws://sockpuppetbrowser:3000
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.changedetection.rule=Host(`changes.example.com`)"
      - "traefik.http.routers.changedetection.entrypoints=https"
      - "traefik.http.routers.changedetection.tls.certresolver=letsencrypt"
      - "traefik.http.services.changedetection.loadbalancer.server.port=5000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

`USE_X_SETTINGS=1` tells the app to trust `X-Forwarded-*` headers from the proxy. Without it, generated links use the internal container name and every link in the UI is wrong.

Keep the `default` network alongside `traefik-net` so the app can still reach the browser container.

## Updating

```bash
cd /opt/docker/changedetection
docker compose pull
docker compose up -d
```

The `:latest` tag here is genuinely the recommended one — this project ships often and the datastore migrates itself forward. [Diun](diun.md) will flag new images if you would rather approve them.

## Backup

Everything is in `/datastore`: the watch list, all history snapshots, and your notification URLs.

```bash
cd /opt/docker/changedetection
docker compose stop changedetection
docker run --rm \
  -v changedetection_changedetection-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/changedetection-$(date +%F).tar.gz -C /data .
docker compose start changedetection
```

There is also a **Backup** button in the UI that produces a zip — easier, and fine for most purposes.

## Troubleshooting

**Page fetches but content is empty.** Client-side rendering. Go to step 6.

**Notification never arrives.** Use the test button. If the test works but real changes stay silent, the watch is probably not actually changing — check the diff history.

**Constant false positives.** The page has a timestamp, a session ID, or rotating ads. Use the visual selector to narrow the scope, or add ignore rules under the watch's **Filters & Triggers** tab. Regex is supported.

**403 or 429 from the target site.** You are polling too fast, or the site blocks datacentre IPs. Slow the interval down first. A custom `User-Agent` under the watch settings sometimes helps.

**`SYS_ADMIN` denied / browser container will not start.** Some hardened hosts and most LXC containers block it. This is one more reason to run Docker in a VM rather than an LXC — see the [Proxmox guide](../proxmox/proxmox-ve-install.md).

**Everything is slow with many watches.** Lower `FETCH_WORKERS`, or raise it if you have RAM to spare — the default is 10.

## Where this sits in my lab

Change Detection runs on the Docker VM on my first Proxmox node with the browser container enabled, and pushes to ntfy so alerts land on my phone.

What it actually watches: firmware pages for hardware that has no update notification, the release notes for a handful of self-hosted apps that do not publish an RSS feed, and one supplier's stock page. It is not glamorous, but it has caught two security updates I would otherwise have missed by a week.
