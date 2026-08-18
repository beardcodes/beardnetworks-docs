# Firefly III

Budgeting apps want your bank credentials, a subscription, and permission to sell your spending data. Firefly III wants a container and a database, and then tells you where the money went.

It is a double-entry accounting system with a friendly interface — which sounds heavier than it is, but that design is why the numbers always reconcile.

[Firefly III](https://www.firefly-iii.org/) is a self-hosted personal finances manager.

1. **Accounts** — current, savings, credit cards, cash, loans, in any currency
2. **Budgets and bills**, with alerts when you overspend or a bill is due
3. **Rules engine** that categorises transactions automatically as they arrive
4. **Reports** — spending by category, net worth over time, income vs expenses
5. **Piggy banks** for savings goals
6. **A full API**, plus a separate Data Importer for bank feeds, CSV and Nordigen/GoCardless

## Three containers

Firefly's Compose stack has parts that look optional and are not:

| Container | Job | Needed? |
|---|---|---|
| `app` | The application | Yes |
| `db` | MariaDB | Yes |
| `cron` | Hits the cron API endpoint nightly | Yes, for recurring transactions and bill reminders |

The `cron` container is a small Alpine image that does nothing but call one URL each night. Skip it and recurring transactions silently never fire, which you notice about six weeks later.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - `openssl` for generating keys
    - A little patience — this is the most configuration-heavy app in this documentation

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/firefly && cd /opt/docker/firefly
sudo nano docker-compose.yml
```

Upstream's stack:

```yaml
services:
  app:
    image: fireflyiii/core:latest
    hostname: app
    container_name: firefly_iii_core
    restart: always
    volumes:
      - firefly_iii_upload:/var/www/html/storage/upload
    env_file: .env
    networks:
      - firefly_iii
    ports:
      - 8088:8080
    depends_on:
      - db

  db:
    image: mariadb:lts
    hostname: db
    container_name: firefly_iii_db
    restart: always
    env_file: .db.env
    networks:
      - firefly_iii
    volumes:
      - firefly_iii_db:/var/lib/mysql

  cron:
    image: alpine
    restart: always
    container_name: firefly_iii_cron
    env_file: .env
    command: ["sh", "-c", "apk add tzdata && \
      (ln -s /usr/share/zoneinfo/$$TZ /etc/localtime || true) && \
      echo \"0 3 * * * wget -qO- http://app:8080/api/v1/cron/$$STATIC_CRON_TOKEN;echo\" \
      | crontab - && \
      crond -f -L /dev/stdout"]
    networks:
      - firefly_iii
    depends_on:
      - app

volumes:
   firefly_iii_upload:
   firefly_iii_db:

networks:
  firefly_iii:
    driver: bridge
```

Upstream publishes on port 80; this uses `8088:8080` since 80 belongs to your reverse proxy.

Note the `$$` in the cron command — the same Compose escaping rule as everywhere else.

## 2. Generate the two keys

Firefly is fussy about lengths, and the errors when you get it wrong are unhelpful.

```bash
# APP_KEY - exactly 32 characters
head /dev/urandom | LC_ALL=C tr -dc 'A-Za-z0-9' | head -c 32; echo

# STATIC_CRON_TOKEN - exactly 32 characters
head /dev/urandom | LC_ALL=C tr -dc 'A-Za-z0-9' | head -c 32; echo
```

!!! danger "Both must be exactly 32 characters"

    Not 31, not 33. `APP_KEY` at the wrong length produces a 500 error on every page with nothing useful in the log. `STATIC_CRON_TOKEN` at the wrong length makes the cron container fail quietly, so recurring transactions stop without any visible sign.

    Do not use `openssl rand -base64 32` — that produces 44 characters.

## 3. The environment files

Two files, because the app and database read different ones.

```bash
sudo nano .env
```

```bash
APP_ENV=production
APP_DEBUG=false
APP_KEY=<your 32-character key>
SITE_OWNER=you@example.com
TZ=Europe/London

DEFAULT_LANGUAGE=en_GB
DEFAULT_LOCALE=equal

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=firefly
DB_USERNAME=firefly
DB_PASSWORD=<a strong password>

STATIC_CRON_TOKEN=<your other 32-character key>

APP_URL=https://firefly.example.com
TRUSTED_PROXIES=**
```

```bash
sudo nano .db.env
```

```bash
MYSQL_RANDOM_ROOT_PASSWORD=yes
MYSQL_USER=firefly
MYSQL_PASSWORD=<the same password as DB_PASSWORD>
MYSQL_DATABASE=firefly
```

```bash
chmod 600 .env .db.env
```

`TRUSTED_PROXIES=**` is needed behind a reverse proxy, or Firefly generates links using the container's internal address and the login form posts to nowhere.

The full reference `.env` from upstream has many more options — start minimal.

## 4. Start it

```bash
docker compose up -d
docker compose logs -f app
```

First boot runs database migrations, which takes a minute or two. Wait for it to settle, then open `http://<host-ip>:8088`.

Register the first account. **The first user registered becomes the owner** — after that, set `DISABLE_FRAME_HEADER` aside and consider whether you want registration open at all.

## 5. Set up accounts first

Before importing anything, create your asset accounts under **Accounts → Asset Accounts**: current account, savings, credit card, cash.

Give each an opening balance and date. Firefly is double-entry, so every transaction moves money *between* accounts — if the accounts do not exist, nothing can be recorded properly, and retrofitting them onto imported data is miserable.

## 6. Import transactions

The importer is a **separate container**, which surprises people:

```yaml
  importer:
    image: fireflyiii/data-importer:latest
    container_name: firefly_iii_importer
    restart: always
    networks:
      - firefly_iii
    ports:
      - 8089:8080
    depends_on:
      - app
    environment:
      FIREFLY_III_URL: http://app:8080
      VANITY_URL: https://firefly.example.com
      FIREFLY_III_ACCESS_TOKEN: ${FIREFLY_TOKEN}
```

Create the token in Firefly under **Options → Profile → OAuth → Personal Access Tokens**.

The importer handles CSV files, and bank connections through Nordigen/GoCardless (Europe) or SimpleFIN (US). CSV works everywhere: download from your bank, map the columns once, save the mapping for reuse.

<!-- screenshot: Firefly III dashboard with account balances and budget progress -->

## 7. Rules — the feature that makes it stick

**Automation → Rules**. A rule has triggers and actions:

- *Description contains "TESCO"* → set category to Groceries
- *Amount is more than 500 and account is Current* → add tag "large"
- *Description contains "SPOTIFY"* → set category Subscriptions, set budget Entertainment

Rules run on new transactions automatically, and can be applied retroactively to existing ones.

Fifteen minutes writing rules turns Firefly from a data-entry chore into something that categorises itself. Without them, you will stop using it within a month.

## 8. Put it behind Traefik

```yaml
    networks:
      - firefly_iii
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.firefly.rule=Host(`firefly.example.com`)"
      - "traefik.http.routers.firefly.entrypoints=https"
      - "traefik.http.routers.firefly.tls.certresolver=letsencrypt"
      - "traefik.http.services.firefly.loadbalancer.server.port=8080"
      - "traefik.docker.network=traefik-net"

networks:
  firefly_iii:
    driver: bridge
  traefik-net:
    external: true
```

Both networks — `traefik-net` for traffic, `firefly_iii` to still reach the database.

Set `APP_URL` to match, and keep `TRUSTED_PROXIES=**`.

Firefly has its own authentication, so no [Tinyauth](tinyauth.md). Given it holds your complete financial history, keep it LAN-only or behind [wg-easy](wg-easy.md).

## Updating

```bash
cd /opt/docker/firefly
docker compose pull
docker compose up -d
```

**Back up first.** Firefly runs database migrations on startup and major versions have required manual steps. Read the [upgrade instructions](https://docs.firefly-iii.org/how-to/firefly-iii/installation/upgrade/) for anything beyond a patch release.

## Backup

```bash
cd /opt/docker/firefly

docker exec firefly_iii_db mysqldump -u firefly -p"$DB_PASSWORD" firefly \
  > /mnt/user/backups/firefly-db-$(date +%F).sql

docker run --rm \
  -v firefly_firefly_iii_upload:/data \
  -v /mnt/user/backups:/backup \
  alpine tar czf /backup/firefly-uploads-$(date +%F).tar.gz -C /data .

cp .env .db.env /mnt/user/backups/
```

The `.env` matters more than it looks — `APP_KEY` is used for encryption, and restoring a database without the matching key leaves you with data you cannot read.

## Troubleshooting

**500 error on every page.** `APP_KEY` is not exactly 32 characters.

**Recurring transactions never fire.** The cron container. `docker logs firefly_iii_cron` — usually `STATIC_CRON_TOKEN` at the wrong length.

**Login redirects back to the login page.** `APP_URL` mismatch, or `TRUSTED_PROXIES` unset behind a proxy.

**"Database not ready" on first start.** MariaDB is still initialising. Wait, then `docker compose restart app`.

**Access denied for user 'firefly'.** `DB_PASSWORD` in `.env` and `MYSQL_PASSWORD` in `.db.env` do not match. If the database volume already exists, changing the file does not change the stored password — you have to update it in MariaDB or start with a fresh volume.

**Importer cannot reach Firefly.** Use `http://app:8080` internally, not the public URL.

**Charts blank or assets 404 behind the proxy.** `APP_URL` and `TRUSTED_PROXIES`.

## Where this sits in my lab

Firefly III runs on `proxmox2`, LAN-only, with the data importer alongside it pulling CSV exports from the bank once a month.

It is the most setup-heavy thing in this documentation — three containers, two env files, two keys of an exact length — and the one I would least want to lose. The rules engine is what made it survive past the first month; before that, categorising transactions by hand was exactly the chore it sounds like.
