# Nginx Proxy Manager

Nginx Proxy Manager is the reverse proxy for people who would rather click than write YAML. Add a host, type a domain, tick "request SSL certificate", done. No labels, no dynamic config files, no restarting anything.

I run it alongside [Traefik](traefik.md), which needs explaining — see the section below before you set this up.

[Nginx Proxy Manager](https://nginxproxymanager.com/) is a web UI on top of nginx and Let's Encrypt.

1. **Proxy hosts in a form** — domain, target IP, port, save
2. **Automatic Let's Encrypt certificates**, including DNS-01 wildcards across dozens of DNS providers
3. **Access lists** — basic auth or IP allow/deny, applied per host
4. **Stream hosts** for raw TCP and UDP forwarding, which Traefik makes considerably harder
5. **Custom nginx config** per host when the UI is not enough
6. **404 hosts and redirections**, handled as first-class objects

## NPM or Traefik?

Both do the same core job. The difference is where the configuration lives:

| | Nginx Proxy Manager | Traefik |
|---|---|---|
| Configuration | Web UI, stored in a database | Labels on containers, or config files |
| Learning curve | Minutes | An afternoon, minimum |
| Fits in version control | No — it is in a database | Yes, it is your Compose file |
| Proxying non-Docker things | Trivial, just type an IP | Needs a file provider entry |
| TCP/UDP streams | Built into the UI | Possible, awkward |
| Middleware ecosystem | Limited | Extensive, including [CrowdSec](crowdsec.md) |

**Traefik** wins when everything you proxy is a container you control, because the routing lives next to the thing it routes to and gets committed alongside it.

**NPM** wins for everything else: a Proxmox web UI on a bare host, a printer, a NAS, a device that only speaks HTTP on a strange port. Things with no Docker labels to attach.

!!! danger "Two reverse proxies cannot share ports 80 and 443"

    If Traefik already binds 80 and 443 on a host, NPM cannot. Your options:

    - **Different hosts** — Traefik on one machine, NPM on another. Cleanest, and what I do.
    - **Different ports** — NPM on 8080/8443, with something in front. Adds a hop and complicates certificates.
    - **Chain them** — Traefik as the edge, forwarding specific hostnames to NPM as a backend.

    Do not attempt to run both on 80/443 on the same host. The second one to start simply fails to bind, and the error is not always obvious.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - Ports 80 and 443 free on this host, and forwarded to it if you want public certificates
    - A domain with DNS you control

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/npm && cd /opt/docker/npm
sudo nano docker-compose.yml
```

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - 80:80        # HTTP
      - 443:443      # HTTPS
      - 81:81        # Admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Newer versions ship with SQLite built in, so the separate MariaDB container older guides insist on is no longer required. Add one only if you are running at a scale where SQLite is genuinely the bottleneck, which in a homelab you are not.

```bash
docker compose up -d
docker compose logs -f
```

First start takes a minute while it initialises the database.

## 2. Log in and change the default credentials

Browse to `http://<host-ip>:81`.

The default login is:

```
Email:    admin@example.com
Password: changeme
```

!!! danger "Change these before anything else"

    These credentials are identical on every NPM install in the world. The UI prompts you to change them on first login — do not skip past it. An admin session here can proxy any internal service to the internet, which makes it a more valuable target than most of what it protects.

It forces you to set a real name, email and password immediately. Use a password manager.

## 3. Point DNS at your server

For a public certificate, the hostname must resolve to your public IP and port 80 must reach NPM. Create an `A` record for each service, or one wildcard:

```
*.example.com   A   <your-public-ip>
```

For internal-only names, add rewrites in [AdGuard](adguard.md) pointing `*.home.lan` at NPM's LAN address instead.

## 4. Add your first proxy host

**Hosts → Proxy Hosts → Add Proxy Host**

**Details** tab:

| Field | Value |
|---|---|
| Domain Names | `uptime.example.com` |
| Scheme | `http` |
| Forward Hostname / IP | `192.168.1.20` |
| Forward Port | `3001` |
| Cache Assets | Off |
| Block Common Exploits | **On** |
| Websockets Support | On |

**Websockets Support** catches people out constantly. Uptime Kuma, Portainer's console, Home Assistant and anything with a live-updating UI need it. Symptom when missing: the page loads, then nothing ever updates.

**SSL** tab:

| Field | Value |
|---|---|
| SSL Certificate | *Request a new SSL Certificate* |
| Force SSL | On |
| HTTP/2 Support | On |
| HSTS Enabled | On, once you are sure it works |

Agree to the Let's Encrypt terms and save. The certificate is issued in a few seconds.

<!-- screenshot: NPM proxy host list showing online status and SSL badges -->

!!! warning "Turn on HSTS last"

    HSTS tells browsers to refuse plain HTTP for that domain, for months. If you enable it before the certificate works properly, you get a domain you cannot reach and cannot easily un-break — clearing it requires each visitor to purge their browser's HSTS cache.

## 5. Get a wildcard certificate

One certificate for `*.example.com` beats managing thirty. This requires the DNS-01 challenge, which proves domain ownership through a DNS record rather than an HTTP request — so it also works for services that are not publicly reachable at all.

**SSL Certificates → Add SSL Certificate → Let's Encrypt**

| Field | Value |
|---|---|
| Domain Names | `*.example.com` and `example.com` |
| Use a DNS Challenge | **On** |
| DNS Provider | Cloudflare, or whoever hosts your DNS |
| Credentials | Your API token |

For Cloudflare, create a scoped API token at **My Profile → API Tokens** with `Zone → DNS → Edit` on that zone only. Do not use the Global API Key — it has full account access, and NPM stores it in its database.

The credentials box wants the file content in the provider's expected format:

```ini
dns_cloudflare_api_token = your_token_here
```

Once issued, every proxy host can select this certificate from the dropdown instead of requesting its own, and internal-only services get valid TLS without ever being exposed.

## 6. Add an access list

**Access Lists → Add Access List** puts basic auth or IP filtering in front of a host.

For an internal tool you want reachable but not public:

- **Authorization** tab: add a username and password
- **Access** tab: `allow 192.168.1.0/24`, `allow 10.42.42.0/24`, then `deny all`

The second rule is the [wg-easy](wg-easy.md) subnet, so VPN clients get through.

Then set the access list on the proxy host's Details tab. Satisfy Any means either auth *or* an allowed IP is enough; leave it off to require both.

## 7. Stream hosts, for non-HTTP services

**Hosts → Streams** forwards raw TCP or UDP. Useful for a game server, a Minecraft instance, or a database you need to reach from elsewhere.

This is the thing NPM does noticeably more easily than Traefik, where TCP routing means entrypoint configuration and TLS passthrough rules.

## Updating

```bash
cd /opt/docker/npm
docker compose pull
docker compose up -d
```

Read the [release notes](https://github.com/NginxProxyManager/nginx-proxy-manager/releases) for major version bumps — database migrations have occasionally needed manual intervention.

## Backup

Everything is in the two bind mounts, which is one of the nicer things about NPM:

```bash
cd /opt/docker/npm
docker compose stop
sudo tar czf /mnt/user/backups/npm-$(date +%F).tar.gz data letsencrypt
docker compose start
```

`data` holds the database, hosts and access lists. `letsencrypt` holds the certificates and account key. Both matter — restoring without the account key means re-issuing everything and possibly hitting rate limits.

## Troubleshooting

**Certificate request fails: "challenge failed".** Port 80 is not reaching NPM from the internet. Check the forward, check the DNS record resolves to your public IP, and check your ISP does not block port 80 — some do. Use the DNS challenge instead if so.

**502 Bad Gateway.** NPM cannot reach the backend. From the container: `docker exec npm curl -I http://192.168.1.20:3001`. Usually a wrong port, or a container that only listens on localhost.

**The site loads but never updates.** Websockets support is off. Step 4.

**Too many certificate requests.** Let's Encrypt rate-limits at 5 duplicate certificates per week. Wait, or use a wildcard so you stop requesting individually.

**Cannot log in after changing the password.** Reset it directly in the database:

```bash
docker exec -it npm sh
# sqlite3 /data/database.sqlite
```

**Admin UI on 81 is unreachable but sites work.** Something else on the host has port 81. `ss -tlnp | grep 81`.

**Real client IPs are not reaching the backend.** NPM forwards `X-Forwarded-For` by default, but the backend has to be configured to trust it — see the `USE_X_SETTINGS` note in the [Change Detection guide](changedetection.md) for an example.

## Where this sits in my lab

NPM runs on `proxmox2`. [Traefik](traefik.md) also lives in this lab but on a different host, precisely because of the port conflict described above.

The split: Traefik handles containers, where labels sit next to the service and get committed with it. NPM handles everything that is not a container — the Proxmox web UIs themselves, the Unraid interface, a couple of appliances that have a web page and nothing else. Typing an IP and a port into a form is genuinely the right tool for those, and writing a Traefik file-provider entry for a printer is not a good use of an evening.
