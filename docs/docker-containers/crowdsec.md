# CrowdSec

Anything you expose to the internet gets scanned within minutes and brute-forced within hours. Fail2ban has handled that for years by reading logs and writing firewall rules. CrowdSec does the same thing, then adds the part fail2ban never had: everyone running it shares what they see, so you block an IP that attacked someone else before it reaches you.

This guide sets up CrowdSec with Traefik, using the bouncer plugin rather than a separate container.

[CrowdSec](https://www.crowdsec.net/) is an open-source, collaborative intrusion prevention system.

1. **Log parsing and scenario detection** — brute force, scanning, credential stuffing, web exploits
2. **Community blocklist** — IPs reported by other CrowdSec users, applied before they touch you
3. **Bouncers** — the enforcement layer, available for Traefik, nginx, iptables, Cloudflare and more
4. **Hub of collections** — pre-written parsers and scenarios for Traefik, SSH, Nextcloud, and most common services
5. **Free tier includes the community blocklist**, which is the main value
6. **Low resource use** — it reads logs, it does not proxy traffic

## The two halves

This trips people up, so get it straight before you start:

| Component | What it does | Where it runs |
|---|---|---|
| **CrowdSec agent / LAPI** | Reads logs, detects attacks, maintains the decision list | Its own container |
| **Bouncer** | Asks the LAPI "is this IP banned?" and blocks accordingly | Inside Traefik, as a plugin |

CrowdSec on its own **blocks nothing**. It detects and records decisions. Without a bouncer you have an alarm with no lock on the door — a surprisingly common misconfiguration.

!!! note "Prerequisites"

    - [Traefik](traefik.md) already running and terminating TLS
    - Traefik configured to write access logs in JSON — covered in step 1
    - Something actually exposed to the internet, or this is a solution without a problem

## 1. Make Traefik log in JSON

CrowdSec parses Traefik's access log, so the log has to exist and be machine-readable. Add to your Traefik static config:

```yaml
      - --accesslog=true
      - --accesslog.filepath=/var/log/traefik/access.log
      - --accesslog.format=json
      - --log.filePath=/var/log/traefik/traefik.log
      - --log.level=INFO
```

And a volume so both containers can reach the file:

```yaml
    volumes:
      - traefik_logs:/var/log/traefik
```

**JSON format is not optional.** CrowdSec's Traefik parser expects it, and the default common-log format silently produces zero detections.

## 2. The CrowdSec container

```yaml
  crowdsec:
    image: crowdsecurity/crowdsec:latest
    container_name: crowdsec
    restart: unless-stopped
    environment:
      - GID=1000
      - COLLECTIONS=crowdsecurity/traefik crowdsecurity/http-cve crowdsecurity/base-http-scenarios crowdsecurity/whitelist-good-actors
      - BOUNCER_KEY_TRAEFIK=${CROWDSEC_BOUNCER_KEY}
    volumes:
      - crowdsec_config:/etc/crowdsec
      - crowdsec_data:/var/lib/crowdsec/data
      - traefik_logs:/var/log/traefik:ro
      - ./acquis.yaml:/etc/crowdsec/acquis.yaml:ro
    networks:
      - traefik-net

volumes:
  crowdsec_config:
  crowdsec_data:
  traefik_logs:
```

`BOUNCER_KEY_TRAEFIK` registers a bouncer with that API key at startup, which saves you generating one by hand. Put a long random string in your `.env`:

```bash
openssl rand -hex 32
```

```bash
echo "CROWDSEC_BOUNCER_KEY=<that string>" >> .env
chmod 600 .env
```

The collections are the detection rules: `traefik` parses the access log, `http-cve` catches known exploit attempts, `base-http-scenarios` covers scanning and enumeration, and `whitelist-good-actors` stops you banning Google's crawler.

## 3. Tell CrowdSec which logs to read

```bash
sudo nano acquis.yaml
```

```yaml
filenames:
  - /var/log/traefik/access.log
labels:
  type: traefik
---
filenames:
  - /var/log/auth.log
labels:
  type: syslog
```

The `type` label selects the parser. `traefik` for the access log; add the host's `auth.log` if you bind-mount it, and SSH brute force gets caught too.

```bash
docker compose up -d crowdsec
docker compose logs -f crowdsec
```

Check it is reading:

```bash
docker exec crowdsec cscli metrics
```

The **Acquisition Metrics** table should show lines being read and parsed. Zeros there mean the log path or the format is wrong, and nothing downstream will work.

## 4. Install the Traefik bouncer plugin

The plugin runs inside Traefik, so there is no extra container. In your Traefik **static** configuration:

```yaml
experimental:
  plugins:
    bouncer:
      moduleName: github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
      version: v1.7.1
```

If you configure Traefik with CLI flags rather than a file:

```yaml
      - --experimental.plugins.bouncer.modulename=github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin
      - --experimental.plugins.bouncer.version=v1.7.1
```

Check the [releases page](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin/releases) for the current version — v1.7.1 is current at the time of writing.

Traefik downloads the plugin at startup, so it needs outbound internet access on first boot after this change.

## 5. Define the middleware

In your Traefik **dynamic** configuration:

```yaml
http:
  middlewares:
    crowdsec:
      plugin:
        bouncer:
          enabled: true
          logLevel: INFO
          crowdsecMode: live
          crowdsecLapiKey: <the same key as BOUNCER_KEY_TRAEFIK>
          crowdsecLapiHost: crowdsec:8080
```

`crowdsecMode` options:

- **`live`** — asks the LAPI on every request. Accurate, adds a millisecond or two. Start here.
- **`stream`** — caches the whole decision list locally and refreshes periodically. Faster, briefly stale.
- **`none`** — plugin disabled but still loaded, useful for testing.

Restart Traefik and watch its logs for the plugin loading.

## 6. Apply it to your routers

The middleware exists but is not doing anything until a router uses it. On any service you expose:

```yaml
    labels:
      - "traefik.http.routers.seerr.middlewares=crowdsec@file"
```

Or apply it to an entire entrypoint, which is better — you cannot forget it on a new service:

```yaml
      - --entrypoints.https.http.middlewares=crowdsec@file
```

!!! danger "Test before you rely on it, and keep a way back in"

    A misconfigured bouncer can ban your own IP and lock you out of everything behind Traefik at once. Before enabling it on the entrypoint:

    1. Add your LAN to the whitelist (step 8)
    2. Confirm you have SSH access that does not pass through Traefik
    3. Know the unban command: `docker exec crowdsec cscli decisions delete --ip <your-ip>`

## 7. Verify it actually blocks

Ban yourself deliberately:

```bash
docker exec crowdsec cscli decisions add --ip 1.2.3.4 --duration 5m --reason "testing"
docker exec crowdsec cscli decisions list
```

Then, using your real IP for a moment:

```bash
docker exec crowdsec cscli decisions add --ip <your-public-ip> --duration 1m --reason "test"
```

Load one of your sites. You should get a 403. After a minute it clears itself.

If you get through, the bouncer is not wired up: check the LAPI key matches on both sides, and that the middleware is actually attached to the router.

<!-- screenshot: cscli decisions list showing active bans with origins -->

## 8. Whitelist your own networks

Nothing is more annoying than banning yourself for testing something.

```bash
docker exec -it crowdsec sh
nano /etc/crowdsec/parsers/s02-enrich/mywhitelists.yaml
```

```yaml
name: me/my-whitelists
description: "Whitelist LAN and VPN"
whitelist:
  reason: "trusted networks"
  cidr:
    - "192.168.1.0/24"
    - "10.42.42.0/24"
```

```bash
docker restart crowdsec
```

The second entry is the [wg-easy](wg-easy.md) subnet, so you never ban yourself while connected over the VPN.

## 9. Enrol in the console, optionally

The free CrowdSec console gives you a dashboard and, more usefully, access to the community blocklist:

```bash
docker exec crowdsec cscli console enroll <your-enrollment-key>
```

Then accept the instance at [app.crowdsec.net](https://app.crowdsec.net/). This is what turns CrowdSec from "fail2ban with better parsers" into something that blocks attackers before they have ever touched your server.

## Useful commands

```bash
# What is currently banned, and why
docker exec crowdsec cscli decisions list

# Alerts, including ones that did not reach ban threshold
docker exec crowdsec cscli alerts list

# Unban
docker exec crowdsec cscli decisions delete --ip 1.2.3.4

# Is it parsing anything?
docker exec crowdsec cscli metrics

# What collections are installed
docker exec crowdsec cscli collections list

# Add a collection
docker exec crowdsec cscli collections install crowdsecurity/nginx
docker restart crowdsec
```

## Updating

```bash
cd /opt/docker/traefik
docker compose pull crowdsec
docker compose up -d crowdsec
docker exec crowdsec cscli hub update && docker exec crowdsec cscli hub upgrade
```

The second command updates the parsers and scenarios, which move independently of the container image. Worth running monthly.

The Traefik plugin version is pinned in your static config — bump it manually and restart Traefik.

## Backup

```bash
docker run --rm -v crowdsec_config:/data -v $(pwd):/backup \
  alpine tar czf /backup/crowdsec-config-$(date +%F).tar.gz -C /data .
```

The data volume holds the decision database, which rebuilds itself. The config volume holds your whitelists and enrolment, which does not.

## Troubleshooting

**`cscli metrics` shows zero lines parsed.** The log file is not being read. Check the path in `acquis.yaml`, that the volume is mounted in both containers, and that Traefik is writing JSON.

**Decisions exist but nothing is blocked.** The bouncer is not working. Check the LAPI key matches exactly, that the middleware is attached to a router or entrypoint, and Traefik's logs for plugin errors.

**Traefik will not start after adding the plugin.** It could not download it. Check outbound connectivity and that the version string is a real tag.

**Everyone gets a 403.** You banned a range, or the whitelist is malformed. `cscli decisions list` and delete the offender.

**All attacks appear to come from one IP.** That is your reverse proxy — you are seeing the proxied address, not the client's. Traefik must be trusted-proxy aware and forwarding `X-Forwarded-For` correctly; if Cloudflare sits in front, set `trustedIPs` for Cloudflare's ranges.

**Banned yourself, locked out.** SSH to the host directly and run the unban command. This is why step 6 says to keep SSH outside Traefik.

## Where this sits in my lab

CrowdSec runs on `proxmox2` next to Traefik, parsing its access log and the host's `auth.log`. The bouncer plugin is applied at the entrypoint, so every service behind Traefik inherits it whether I remember to think about it or not.

The honest assessment: almost nothing of mine is exposed — [wg-easy](wg-easy.md) handles remote access and everything else is LAN-only. The one public service is [Seerr](seerr.md). CrowdSec exists to protect that one door, and the community blocklist means most of the scanning traffic never gets a response at all. The logs are genuinely educational: the volume of automated attacks against a small residential IP is not something you appreciate until you can see it.
