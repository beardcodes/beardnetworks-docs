# Vaultwarden

A password manager is the one self-hosted service where getting it wrong has consequences beyond your own inconvenience. It is also the one most worth self-hosting, because the alternative is trusting someone else with every credential you own.

Vaultwarden is a Bitwarden-compatible server that runs in about 100 MB of RAM. The official Bitwarden server needs several containers and an SQL Server; this is one binary and a SQLite file, and every official Bitwarden client works with it.

[Vaultwarden](https://github.com/dani-garcia/vaultwarden) is an unofficial Bitwarden-compatible server written in Rust, formerly known as bitwarden_rs.

1. **Works with all official clients** — browser extensions, mobile apps, desktop, CLI
2. **Tiny** — a single container, minimal memory, runs happily on a Raspberry Pi
3. **Premium features included** — TOTP, attachments, Bitwarden Send, Emergency Access
4. **Organisations and collections** for sharing credentials with family
5. **Admin panel** for user management and diagnostics
6. **Argon2 hashing**, WebAuthn, and hardware key support

!!! danger "Read this before you deploy"

    This container will hold every password you own. Three rules, and they are not optional:

    1. **HTTPS is mandatory.** The Bitwarden clients refuse to connect over plain HTTP to anything but localhost. This is not a limitation to work around — it is the clients protecting you.
    2. **Back it up, and test the restore.** Losing this volume means losing every credential with no recovery path. There is no "forgot password" for a zero-knowledge vault.
    3. **Turn off public signups** immediately, or anyone who can reach the page can register on your server.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - A reverse proxy with a valid certificate — [Traefik](traefik.md) or [NPM](nginx-proxy-manager.md)
    - A domain name, even for internal-only use

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/vaultwarden && cd /opt/docker/vaultwarden
sudo nano docker-compose.yml
```

Upstream's baseline:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vw.example.com"
    volumes:
      - ./vw-data/:/data/
    ports:
      - 127.0.0.1:8000:80
```

Note the bind: **`127.0.0.1:8000:80`**, not `8000:80`. Upstream deliberately binds to localhost only, so the service is unreachable from the network until you put a proxy in front of it with TLS. Leave that as it is.

`DOMAIN` must be the exact public URL, HTTPS included. It is used for generating links, WebAuthn and attachment URLs — a mismatch breaks two-factor registration in confusing ways.

## 2. Generate an admin token

The admin panel should be protected by an Argon2 hash, not a plaintext string:

```bash
docker run --rm -it vaultwarden/server /vaultwarden hash
```

It prompts for a password and prints a hash starting `$argon2id$`.

!!! danger "Double the dollar signs in the Compose file"

    Like [Tinyauth](tinyauth.md), an Argon2 hash is full of `$`, and Compose will interpolate them. Every `$` becomes `$$`, or use an `.env` file where no escaping is needed.

Add to `.env`:

```bash
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'
```

```bash
chmod 600 .env
```

Then reference it:

```yaml
    environment:
      DOMAIN: "https://vw.example.com"
      ADMIN_TOKEN: ${ADMIN_TOKEN}
      SIGNUPS_ALLOWED: "false"
```

Leaving `ADMIN_TOKEN` unset disables the admin panel entirely, which is a legitimate choice if you do not need it.

## 3. Full configuration

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vw.example.com"
      ADMIN_TOKEN: ${ADMIN_TOKEN}
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ALLOWED: "true"
      SIGNUPS_DOMAINS_WHITELIST: "example.com"
      PUSH_ENABLED: "false"
      LOG_LEVEL: "warn"
    volumes:
      - ./vw-data/:/data/
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vaultwarden.rule=Host(`vw.example.com`)"
      - "traefik.http.routers.vaultwarden.entrypoints=https"
      - "traefik.http.routers.vaultwarden.tls.certresolver=letsencrypt"
      - "traefik.http.services.vaultwarden.loadbalancer.server.port=80"

networks:
  traefik-net:
    external: true
```

What the settings do:

1. **`SIGNUPS_ALLOWED: "false"`** — the single most important line. Without it your password server is open registration.
2. **`INVITATIONS_ALLOWED: "true"`** — you can still invite family through the admin panel.
3. **`SIGNUPS_DOMAINS_WHITELIST`** — belt and braces if you ever re-enable signups.
4. The `ports:` block is gone once Traefik is handling it.

Modern Vaultwarden serves WebSocket notifications on the same port 80, so no second entrypoint or `/notifications/hub` routing is needed — older guides that add one are out of date.

```bash
docker compose up -d
docker compose logs -f
```

## 4. Create your account

Browse to `https://vw.example.com`.

With signups disabled you cannot register directly. Two options:

- Temporarily set `SIGNUPS_ALLOWED: "true"`, create your account, set it back and recreate the container. Simplest.
- Use the admin panel at `https://vw.example.com/admin` to invite yourself.

**Your master password cannot be recovered.** Not by you, not by me, not by anyone with root on the host — that is the point of the design. Write it down and put it somewhere physical.

<!-- screenshot: Vaultwarden web vault after first login -->

## 5. Turn on two-factor

**Settings → Security → Two-step Login**. A TOTP app or a hardware key.

There is a bootstrapping problem worth thinking about: if your TOTP codes live in Vaultwarden, and Vaultwarden requires TOTP to open, you have built a locked box containing its own key. Keep the Vaultwarden 2FA seed in a different authenticator app, and save the recovery code somewhere offline.

## 6. Connect the clients

Every official Bitwarden client works. In the browser extension or mobile app, before logging in, tap the settings gear and set the **server URL** to `https://vw.example.com`.

That step is easy to miss and produces a confusing "invalid credentials" error, because the client is cheerfully asking bitwarden.com about an account that does not exist there.

## 7. Backup — the part that actually matters

The vault is encrypted at rest with your master password, so a backup is safe to store, but losing it is unrecoverable.

```bash
#!/bin/bash
# /opt/docker/vaultwarden/backup.sh
set -euo pipefail
BACKUP_DIR=/mnt/user/backups/vaultwarden
STAMP=$(date +%F)
mkdir -p "$BACKUP_DIR"

# SQLite must be backed up with .backup, not cp
docker exec vaultwarden sqlite3 /data/db.sqlite3 ".backup '/data/backup.sqlite3'"

tar czf "$BACKUP_DIR/vaultwarden-$STAMP.tar.gz" \
  -C /opt/docker/vaultwarden/vw-data \
  backup.sqlite3 attachments sends config.json rsa_key.pem 2>/dev/null || true

docker exec vaultwarden rm -f /data/backup.sqlite3
find "$BACKUP_DIR" -name "vaultwarden-*.tar.gz" -mtime +30 -delete
```

```bash
chmod +x backup.sh
sudo crontab -e
# 0 3 * * * /opt/docker/vaultwarden/backup.sh
```

Three things this gets right that a naive `cp` does not:

- **`sqlite3 .backup`** produces a consistent snapshot of a live database. Copying `db.sqlite3` while it is in use can yield a corrupt file that looks fine until you need it.
- **`rsa_key.pem`** is included. Without it, restored data cannot be decrypted properly, and plenty of people have discovered this at the worst moment.
- **Attachments and sends** are separate from the database.

Add a [push monitor in Uptime Kuma](uptime-kuma.md#4-pick-the-right-monitor-type) so you find out when the backup stops running.

**Test the restore.** Restore into a throwaway container and log in. A backup you have never restored is a hypothesis.

## Updating

```bash
cd /opt/docker/vaultwarden
docker compose pull
docker compose up -d
```

Back up first. Vaultwarden tracks upstream Bitwarden's API, and a client update occasionally requires a server update to match — worth keeping reasonably current rather than years behind.

## Troubleshooting

**Clients refuse to connect.** HTTPS with a valid certificate is required. Self-signed will not work on mobile without installing the CA.

**"An error has occurred" on login.** `DOMAIN` does not match the URL you are using.

**Cannot reach /admin.** `ADMIN_TOKEN` is unset, or the `$$` escaping mangled the hash. Check the logs.

**2FA registration fails.** Almost always `DOMAIN`. WebAuthn binds to the exact origin.

**Websocket / live sync not working.** Modern versions need no special routing; confirm your proxy passes WebSocket upgrade headers. On [NPM](nginx-proxy-manager.md) that is the **Websockets Support** toggle.

**Attachments upload but do not download.** Check the `attachments` directory permissions inside `vw-data`.

**Database is locked.** SQLite on network storage. Move `vw-data` to local disk — the same rule as [Mealie](mealie.md).

## Where this sits in my lab

Vaultwarden runs on `proxmox2` behind Traefik with a real certificate. It is one of very few services here reachable from outside the LAN, because a password manager you cannot reach on your phone in a shop is a password manager you will stop using.

That exposure is why it sits behind [CrowdSec](crowdsec.md), has signups disabled, and enforces 2FA. Backups run nightly to the Unraid box and weekly to offsite storage, with a push monitor in Uptime Kuma watching that the job ran.

It is the one service in this documentation where I have actually tested the restore, because it is the one where finding out the backup was broken would be genuinely serious.
