# Uptime Kuma

A homelab has a specific failure mode: something breaks quietly and you find out three days later when you go to use it. Uptime Kuma is the fix. It checks everything on a schedule and tells you the moment something stops answering.

It is the first thing I would install on a new lab after Docker itself.

[Uptime Kuma](https://uptime.kuma.pet/) is a self-hosted monitoring tool — a Pingdom or UptimeRobot you run yourself, with no check limits.

1. **Many monitor types** — HTTP, TCP, ping, DNS, Docker container, database queries, push
2. **90+ notification providers** via a built-in integration list: ntfy, Discord, Telegram, email, Gotify, webhooks
3. **Status pages**, public or private, with your own domain
4. **Certificate expiry monitoring**, with warnings before it bites
5. **Maintenance windows**, so planned downtime does not page you
6. **Genuinely good-looking**, which matters for a thing you glance at daily

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/uptime-kuma && cd /opt/docker/uptime-kuma
sudo nano docker-compose.yml
```

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - uptime-kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - 3001:3001

volumes:
  uptime-kuma:
```

Notes:

1. **`:1`, not `:latest`.** Version 2 is in beta and changes the database backend. Pin the major version and move deliberately.
2. **The Docker socket is mounted read-only** and is optional — it enables the "Docker Container" monitor type, which checks whether a container is actually running rather than whether a port answers. Drop the line if you would rather not mount the socket at all; everything else still works.
3. Port 3001 is unusual enough to rarely conflict.

```bash
docker compose up -d
```

## 2. Create the admin account

Browse to `http://<host-ip>:3001`. The first screen sets up the admin user. There is no default password to change, which is a nice touch.

## 3. Add your first monitor

**Add New Monitor**:

| Field | Value |
|---|---|
| Monitor Type | HTTP(s) |
| Friendly Name | `Seerr` |
| URL | `http://192.168.1.50:5055` |
| Heartbeat Interval | 60 seconds |
| Retries | 3 |
| Accepted Status Codes | `200-299` |

**Retries matters more than it looks.** With retries at 0, a single slow response at 3am sends you an alert. At 3, a service has to fail three consecutive checks before it counts as down — which filters out nearly all the noise without meaningfully delaying real alerts.

## 4. Pick the right monitor type

The default HTTP check is not always what you want:

| Type | Checks | Good for |
|---|---|---|
| HTTP(s) | An HTTP response | Web apps |
| TCP Port | Port accepts a connection | Databases, SSH, game servers |
| Ping | ICMP reply | Switches, APs, printers |
| DNS | A record resolves | Your [AdGuard](adguard.md) instance |
| Docker Container | Container is running | Things with no network listener |
| HTTP(s) - Keyword | Response *contains* specific text | Apps that return 200 while broken |
| Push | Your script calls Kuma | Backups, cron jobs |

**HTTP - Keyword** is the underrated one. Plenty of apps return a cheerful `200 OK` on a page that says "database connection failed". Checking for a keyword you expect to see on a working page catches that; a status code check does not.

**Push** is the other one worth knowing. Kuma gives you a URL, and your script calls it on success:

```bash
#!/bin/bash
restic backup /mnt/user/data && curl -fsS "http://192.168.1.20:3001/api/push/AbC123?status=up&msg=OK"
```

If the backup fails, the URL is never called, and Kuma alerts because the push did not arrive. This turns "did my backup run?" from a thing you hope about into a thing you get told about.

<!-- screenshot: Uptime Kuma dashboard with a grid of monitors and uptime percentages -->

## 5. Set up notifications

**Settings → Notifications → Setup Notification**

[ntfy](https://ntfy.sh/) is the easiest self-hosted option — no account, no webhook registration, works on iOS and Android:

| Field | Value |
|---|---|
| Notification Type | `ntfy` |
| Server URL | `https://ntfy.example.com` |
| Topic | `homelab` |
| Priority | 5 for down, so it breaks through Do Not Disturb |

Tick **Default enabled** and **Apply on all existing monitors**, or you will add a monitor next month, forget the notification, and learn nothing when it fails.

Hit **Test**. An untested notification channel is not a notification channel.

## 6. Monitor certificate expiry

On any HTTPS monitor, **Certificate Expiry Notification** warns you before a certificate lapses.

If your certificates come from Let's Encrypt via [Traefik](traefik.md) or [NPM](nginx-proxy-manager.md) they should auto-renew — but "should" is exactly the assumption this catches. Renewal breaks silently more often than you would expect, usually because a DNS API token expired.

## 7. Build a status page

**Status Pages → New Status Page**.

Add monitors, group them, set a custom domain if you want. Two genuine uses:

- **Public**, for the people you share media with, so they can check before messaging you
- **Private**, as a homelab dashboard that loads faster than the main UI

You can pin it to `/status` and give it your own logo and description.

## 8. What to actually monitor

Resist monitoring everything. A dashboard with forty red squares gets ignored. Monitor what you would want to be woken for, and layer it:

**Infrastructure first** — these failing explains everything else:

- Proxmox hosts, by ping
- The Unraid box, by ping
- Your router
- [AdGuard](adguard.md), by DNS query type — a dead DNS server takes the whole house down and is the single highest-value check on this list

**Then the services people notice:**

- Plex and Jellyfin
- [Seerr](seerr.md)
- Home Assistant

**Then the plumbing:**

- Traefik and NPM
- The *arr stack, by HTTP
- Gitea

**Then the jobs**, via push monitors: backups, snapshot syncs, anything on a cron.

!!! warning "Do not run Uptime Kuma on the host it is monitoring"

    If Kuma lives on the same machine as everything it watches, a host failure takes down both the services and the thing that would have told you. Put it on a different node — ideally the one that does the least.

## Updating

```bash
cd /opt/docker/uptime-kuma
docker compose pull
docker compose up -d
```

Staying on the `:1` tag keeps you on version 1.x. Moving to 2.x is a deliberate migration, not a `pull`.

## Backup

```bash
docker compose stop
docker run --rm \
  -v uptime-kuma_uptime-kuma:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/uptime-kuma-$(date +%F).tar.gz -C /data .
docker compose start
```

Stop it first — SQLite again. There is also **Settings → Backup** in the UI, though upstream considers it deprecated in favour of copying the data directory.

## Troubleshooting

**Monitor shows down but the service works in a browser.** Kuma is checking from inside a container. `docker exec uptime-kuma curl -I <url>`. Usually DNS, or a hostname that only resolves on your workstation.

**Certificate errors on internal HTTPS services.** Self-signed certificates fail validation. Turn off **Ignore TLS/SSL error** — that is, turn the ignore *on* — for that monitor.

**Notifications not arriving.** Test the notification directly from Settings. If the test works but real alerts do not, the notification is not attached to the monitor.

**Alert storms at the same time every night.** Something is restarting — a backup job, a log rotation, a scheduled Docker restart. Use a maintenance window rather than lowering the check frequency.

**Docker container monitors all show down.** The socket mount is missing, or permissions block it. `docker exec uptime-kuma ls -l /var/run/docker.sock`.

**High CPU with many monitors.** Intervals are too aggressive. 60 seconds is fine for most things; infrastructure can sit at 20, and a status page for friends does not need 10.

**Uptime percentages look wrong after a restart.** Kuma counts the time it was down as unknown, not up. It settles.

## Where this sits in my lab

Uptime Kuma runs on `proxmox2`, deliberately away from the Unraid box and the first Proxmox node, so it survives to report their failures.

It watches both hypervisors, the Unraid array, AdGuard by DNS query, and about twenty services by HTTP — plus push monitors for the nightly backups. Notifications go to ntfy at priority 5, which is loud enough to matter and rare enough that when the phone buzzes I actually look.

The AdGuard DNS check has earned its keep more than anything else here. When DNS dies, everything in the house appears broken simultaneously and the cause is not obvious from the symptoms.
