# HortusFox

Every houseplant has a watering schedule you will forget, a species name you half-remember, and a slow decline you only notice once it is well underway. HortusFox is a plant journal that fixes the forgetting part: an inventory with photos, watering intervals, task reminders and a history of what you did to each plant.

It is a small, cheerful application that solves a real problem, which is the best category of self-hosted software.

[HortusFox](https://github.com/danielbrendel/hortusfox-web) is a self-hosted collaborative plant management and tracking system.

1. **Plant inventory** with photos, locations, and per-plant notes
2. **Watering and fertilising intervals**, with a dashboard of what is due
3. **Task list** and a calendar for anything on a schedule
4. **History log** per plant, so you can see what you changed before it started sulking
5. **Collaborative** — multiple users sharing one inventory, with a chat
6. **Environmental tracking** — temperature and humidity per location, optionally fed from sensors

## Two containers

Unlike most things in this section, HortusFox needs a database. It is a PHP application backed by MariaDB.

| Container | Job |
|---|---|
| `app` | The web application |
| `db` | MariaDB, holding everything |

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - A port for the web UI — upstream's example uses 8080, which collides with plenty

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/hortusfox && cd /opt/docker/hortusfox
sudo nano docker-compose.yml
```

Upstream's file, with the credentials moved out to `.env` and the database no longer published to the host:

```yaml
services:
  app:
    image: ghcr.io/danielbrendel/hortusfox-web:latest
    container_name: hortusfox
    restart: unless-stopped
    ports:
      - "8085:80"
    volumes:
      - app_images:/var/www/html/public/img
      - app_attachments:/var/www/html/public/attachments
      - app_logs:/var/www/html/app/logs
      - app_backup:/var/www/html/public/backup
      - app_themes:/var/www/html/public/themes
      - app_migrate:/var/www/html/app/migrations
    environment:
      APP_ADMIN_EMAIL: ${ADMIN_EMAIL}
      APP_ADMIN_PASSWORD: ${ADMIN_PASSWORD}
      APP_TIMEZONE: "Europe/London"
      DB_HOST: db
      DB_PORT: 3306
      DB_DATABASE: hortusfox
      DB_USERNAME: hortusfox
      DB_PASSWORD: ${DB_PASSWORD}
      DB_CHARSET: "utf8mb4"
    depends_on:
      - db

  db:
    image: mariadb
    container_name: hortusfox-db
    restart: always
    environment:
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MARIADB_DATABASE: hortusfox
      MARIADB_USER: hortusfox
      MARIADB_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
  app_images:
  app_attachments:
  app_logs:
  app_backup:
  app_themes:
  app_migrate:
```

Two changes from upstream worth knowing about:

!!! danger "Upstream's example publishes MariaDB on port 3306"

    The published example includes `ports: - "3306:3306"` on the database, and default passwords of `password` and `my-secret-pw`. That combination puts a MariaDB instance with a guessable password on your network.

    The app reaches the database over the internal Docker network by hostname — it does not need a published port. **Remove it**, as above, unless you specifically want external database access. Same reasoning as any other database container.

The port is also moved to `8085:80` here, since 8080 is heavily contested by [Glance](glance.md), [qBittorrent](qbittorrent.md) and cAdvisor.

## 2. Generate the secrets

```bash
sudo nano .env
```

```bash
ADMIN_EMAIL=you@example.com
ADMIN_PASSWORD=<a real password>
DB_PASSWORD=<openssl rand -base64 24>
DB_ROOT_PASSWORD=<openssl rand -base64 24>
```

```bash
chmod 600 .env
```

`APP_ADMIN_EMAIL` and `APP_ADMIN_PASSWORD` create the first account on initial startup. They are read once, at database initialisation — changing them later in the file does nothing.

## 3. Start it

```bash
docker compose up -d
docker compose logs -f app
```

First boot runs migrations against the empty database, which takes a moment. Wait for the app to settle before loading the page, or you will get a database error that resolves itself.

Browse to `http://<host-ip>:8085` and log in with the admin email and password from `.env`.

## 4. Set up locations first

Before adding plants, create the places they live: **Locations**. A windowsill, a bathroom, a greenhouse, a desk.

This is worth doing properly at the start because everything else hangs off it — plants belong to locations, the environmental tracking is per location, and the "what needs watering" view groups by it. Retrofitting locations onto forty plants is tedious.

## 5. Add plants

**Add Plant**, with a photo, name, species and location.

The fields that make it useful later:

| Field | Why it matters |
|---|---|
| Watering interval | Drives the "due" dashboard, which is the whole point |
| Fertilising interval | Same, on a longer cycle |
| Perennial / annual | Affects what shows in seasonal views |
| Notes | Where it came from, how it behaves, what killed the last one |

<!-- screenshot: HortusFox plant inventory grouped by location -->

## 6. Use the task list

**Tasks** covers anything that is not watering — repotting, pruning, treating for pests. Assign to a user and a date.

The history log per plant is the sleeper feature. When a plant starts declining, the log tells you what changed and when, which is the difference between diagnosing a problem and guessing at it.

## 7. Put it behind Traefik

```yaml
    networks:
      - default
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.hortusfox.rule=Host(`plants.home.lan`)"
      - "traefik.http.routers.hortusfox.entrypoints=https"
      - "traefik.http.routers.hortusfox.tls.certresolver=letsencrypt"
      - "traefik.http.services.hortusfox.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

The app container needs **both** networks — `traefik-net` for incoming traffic and `default` to still reach the database. Attaching only `traefik-net` gives you a reachable app that cannot connect to MariaDB.

The service port is **80**, the container's port, not the 8085 published on the host.

HortusFox has its own accounts, so no [Tinyauth](tinyauth.md) needed.

## Updating

```bash
cd /opt/docker/hortusfox
docker compose pull
docker compose up -d
```

Migrations run automatically at startup. Watch the app logs on the first boot after an upgrade, and take the backup below first — a failed migration against a database with no backup is a bad afternoon.

## Backup

Both halves matter: the database holds the records, the volumes hold the photos.

```bash
cd /opt/docker/hortusfox

# Database - use mysqldump, not a file copy of a running database
docker exec hortusfox-db mysqldump -u root -p"$DB_ROOT_PASSWORD" hortusfox \
  > /mnt/user/backups/hortusfox-db-$(date +%F).sql

# Images and attachments
docker run --rm \
  -v hortusfox_app_images:/img \
  -v hortusfox_app_attachments:/att \
  -v /mnt/user/backups:/backup \
  alpine tar czf /backup/hortusfox-files-$(date +%F).tar.gz -C / img att
```

Use `mysqldump` rather than copying `/var/lib/mysql` from under a running server. There is also a backup area inside the app itself, mounted at `app_backup`.

## Troubleshooting

**"Connection refused" or a database error on first load.** MariaDB is still initialising. Give it a minute; `docker compose logs db` will show when it is ready to accept connections.

**Admin login rejected.** `APP_ADMIN_*` is only read when the database is first created. If you changed it afterwards, either reset the password inside the app or drop the database volume and start over — the latter deletes everything.

**App loads but has no styling, or images 404.** Reverse proxy configuration. Check the app knows its own external URL and that you are proxying to port 80.

**Cannot reach the database after adding Traefik.** The `default` network is missing from the app container. Step 7.

**Uploads fail.** Volume permissions on `app_images`, or a PHP upload size limit for a large photo.

**Everything vanished after `docker compose down -v`.** `-v` deletes volumes. That flag has ruined a lot of people's day; there is no recovery without a backup.

## Where this sits in my lab

HortusFox runs on the Docker VM on my first Proxmox node, LAN-only behind Traefik, with the database not published anywhere.

It is the least infrastructural thing in this documentation and gets opened more often than most of the rest. The watering dashboard is the bit that earns it — the plants that used to die were not dying of neglect exactly, they were dying of me being confident I had watered them more recently than I had.
