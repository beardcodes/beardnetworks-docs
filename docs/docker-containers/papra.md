# Papra

Paper arrives, gets photographed or scanned, and lands in a folder called `Scans` where it is never found again. Papra is a document archive: throw files at it, and it makes them searchable, tagged and organised.

It positions itself as the minimalistic option in a category that also contains Paperless-ngx — a deliberate trade of features for a much simpler deployment.

[Papra](https://papra.app/) is a minimalistic document archiving platform.

1. **Full-text search** across your documents
2. **Tags and organisations** for structure, with multi-tenant separation
3. **Tagging rules** that apply tags automatically based on content
4. **Drag-and-drop ingestion**, plus an intake folder for automated imports
5. **Single container** — SQLite and local file storage, no separate database or search engine
6. **Rootless image**, which is a nicer default than most

## Papra or Paperless-ngx?

Both archive documents. The difference is scope:

| | Papra | Paperless-ngx |
|---|---|---|
| Containers | 1 | 3–4 (app, database, Redis, sometimes Tika) |
| OCR | Basic | Extensive, with language packs |
| Maturity | Younger | Long-established, very large community |
| Resource use | Light | Noticeably heavier |
| Setup time | Minutes | An afternoon |

If you are digitising decades of paper and need serious OCR, Paperless-ngx is the stronger tool. If you want somewhere sensible to put PDFs that are already text — invoices, statements, warranties, manuals — Papra does that with a fraction of the operational weight.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - `openssl` for the auth secret

## 1. Create the data directories

The rootless image runs as your user, so the directories need to exist with the right ownership first:

```bash
sudo mkdir -p /opt/docker/papra && cd /opt/docker/papra
mkdir -p ./papra-data/{db,documents}
```

## 2. Generate the auth secret

```bash
openssl rand -hex 48
```

## 3. The Compose file

Upstream documents `docker run`; this is the equivalent Compose:

```yaml
services:
  papra:
    image: ghcr.io/papra-hq/papra:latest
    container_name: papra
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - 1221:1221
    volumes:
      - ./papra-data:/app/app-data
    environment:
      APP_BASE_URL: http://192.168.1.20:1221
      AUTH_SECRET: ${AUTH_SECRET}
```

```bash
echo "AUTH_SECRET=<your 96-character hex string>" > .env
chmod 600 .env
```

Notes:

1. **`user: "1000:1000"`** should match the owner of `papra-data`. Run `id -u` and `id -g` to check. On Unraid use `99:100`.
2. **`APP_BASE_URL`** must be the address you actually browse to. Wrong value breaks login redirects and share links.
3. **Port 1221** is unusual enough to avoid every collision in this documentation.
4. Everything — database and documents — lives under `/app/app-data`, so there is exactly one thing to back up.

```bash
docker compose up -d
docker compose logs -f
```

## 4. Create your account

Browse to `http://<host-ip>:1221` and register. The first account is yours.

Once created, consider disabling registration so the instance is not open — check the [configuration docs](https://docs.papra.app/self-hosting/configuration/) for the current variable name, as this project is moving quickly.

## 5. Create an organisation

Papra groups documents into organisations. Even for personal use, create one — it is the container everything else lives in, and documents cannot exist outside it.

For a household, one organisation per person keeps things separate while sharing an instance.

## 6. Ingest documents

Two routes:

**Drag and drop** into the web UI. Fine for occasional additions.

**The intake folder** is the better one for anything regular. Papra watches a directory and imports what appears in it. Mount a share your scanner writes to:

```yaml
    volumes:
      - ./papra-data:/app/app-data
      - /mnt/user/scans:/app/intake
```

Then point your scanner or phone app at that share, and documents file themselves. Check the current documentation for the intake configuration variable — this is a newer feature and the name has changed.

<!-- screenshot: Papra document list with tags and search -->

## 7. Tagging rules

**Settings → Tagging rules** applies tags automatically based on document content — a document containing "British Gas" gets the `utilities` tag, one containing "Invoice" gets `invoices`.

Set these up before bulk-importing. Rules applied to 400 documents at once is a much better experience than tagging 400 documents by hand, and the temptation is always to import first and organise later.

## 8. Put it behind Traefik

```yaml
    environment:
      APP_BASE_URL: https://docs.example.com
      AUTH_SECRET: ${AUTH_SECRET}
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.papra.rule=Host(`docs.example.com`)"
      - "traefik.http.routers.papra.entrypoints=https"
      - "traefik.http.routers.papra.tls.certresolver=letsencrypt"
      - "traefik.http.services.papra.loadbalancer.server.port=1221"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

**Change `APP_BASE_URL` at the same time.** Papra has its own accounts, so no [Tinyauth](tinyauth.md) needed — but it holds your bank statements and contracts, so keep it LAN-only or behind [wg-easy](wg-easy.md).

## Updating

```bash
cd /opt/docker/papra
docker compose pull
docker compose up -d
```

Papra is a young project and moves fast. Pin a version tag rather than `latest` once you have data in it, read the release notes, and back up before upgrading. The convenience of `latest` is not worth an unattended schema migration on your document archive.

## Backup

One directory, which is the nicest thing about this deployment:

```bash
cd /opt/docker/papra
docker compose stop
sudo tar czf /mnt/user/backups/papra-$(date +%F).tar.gz papra-data .env
docker compose start
```

Stop it first — SQLite. The archive contains the database *and* the documents, so a restore is a single extract.

The documents are stored as regular files under `papra-data/documents`, which is worth knowing: even if Papra itself became unusable, your files are still there in a normal filesystem. That is a meaningful property for an archive you intend to keep for years, and one reason to prefer it over systems that store blobs in a database.

## Troubleshooting

**Permission denied on startup.** The `user:` value does not match the owner of `papra-data`. `ls -ln papra-data` and compare, then `chown -R`.

**Login loops or redirects to the wrong host.** `APP_BASE_URL` does not match the address in your browser.

**"Invalid auth secret" or sessions dropping.** `AUTH_SECRET` changed, or is unset. Changing it logs everyone out.

**Uploads fail for large files.** Reverse proxy body size limit. Traefik does not limit by default; [NPM](nginx-proxy-manager.md) does — raise `client_max_body_size` in the host's advanced config.

**Search returns nothing for a document you can see.** It is a scanned image with no text layer. Papra's OCR is basic; run the PDF through an OCR tool first, or use Paperless-ngx if this is your main use case.

**Documents visible but not downloadable.** File permissions inside `papra-data/documents`.

## Where this sits in my lab

Papra runs on the Docker VM on my first Proxmox node, LAN-only, with an intake folder on the Unraid box that the scanner writes to.

It handles the documents that are born digital — invoices, statements, warranties, manuals — where there is already a text layer and the only real problem is findability. That is most of what arrives now, and it needs neither the OCR pipeline nor the four containers that a heavier system would bring with it.
