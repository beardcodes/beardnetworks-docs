# Mealie

Recipe websites are 2,000 words of childhood anecdote wrapped around a list of ingredients, plus a video that autoplays. Mealie takes a URL, throws all of that away, and keeps the recipe.

Then it does the genuinely useful part: meal planning and a shopping list that adds up the ingredients across everything you planned.

[Mealie](https://mealie.io/) is a self-hosted recipe manager and meal planner with a REST API backend and a Vue frontend.

1. **Import by URL** — most recipe sites publish structured data, and Mealie reads it
2. **Meal planner** — assign recipes to days, with a shopping list generated from the plan
3. **Shopping lists** that consolidate quantities and group by supermarket aisle
4. **Multi-user with households**, so everyone gets their own plan and list
5. **OCR and manual entry** for recipes that came off paper
6. **A real API**, which means [Home Assistant](https://www.home-assistant.io/) can display tonight's dinner on a wall tablet

## SQLite or Postgres?

Mealie ships with SQLite and that is fine for a household. One caveat from upstream matters here:

!!! danger "Do not put a SQLite database on network storage"

    Mealie's documentation is explicit: if the data directory lives on a NAS or any network-attached share, use Postgres instead. SQLite over NFS or SMB causes database corruption and locking errors.

    On Unraid that means: if the appdata share is on the array or a network mount, either move it to a local cache pool, or run Postgres. This applies to most SQLite apps, but Mealie is one where people hit it often, because recipe libraries feel like something to keep with the media.

This guide uses SQLite with the data on local storage.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - Local (not network) storage for the data volume

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/mealie && cd /opt/docker/mealie
sudo nano docker-compose.yml
```

Upstream's SQLite deployment:

```yaml
services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:v3.22.0
    container_name: mealie
    restart: always
    ports:
      - "9925:9000"
    deploy:
      resources:
        limits:
          memory: 1000M
    volumes:
      - mealie-data:/app/data/
    environment:
      ALLOW_SIGNUP: "false"
      PUID: 1000
      PGID: 1000
      TZ: Europe/London
      BASE_URL: https://mealie.example.com

volumes:
  mealie-data:
```

Four things worth understanding:

1. **Pin the version.** Upstream explicitly advises against `latest` — check the current release badge on the [repository](https://github.com/mealie-recipes/mealie) and set it deliberately. `v3.22.0` is what their docs show at the time of writing.
2. **The memory limit is recommended, not optional.** Python pre-allocates generously on a machine with plenty of RAM, and Mealie will happily idle at a high figure without it.
3. **`9925:9000`** — the container listens on 9000; the host port is arbitrary.
4. **`ALLOW_SIGNUP: "false"`** — otherwise anyone reaching the page can create an account. Invite users deliberately instead.

On Unraid, set `PUID: 99` and `PGID: 100` as with the [media stack](media-stack.md#puid-pgid-and-umask).

```bash
docker compose up -d
docker compose logs -f
```

## 2. First login

Browse to `http://<host-ip>:9925`.

The default credentials are:

```
Email:    changeme@example.com
Password: MyPassword
```

!!! danger "Change these immediately"

    They are the same on every Mealie install. Go to the user profile and change both the email and the password before doing anything else.

## 3. Import your first recipe

**Recipes → Create → Import with URL**, paste a link, and Mealie fetches it.

It works by reading the schema.org structured data most recipe sites publish for search engines. Where that exists, the import is clean — ingredients, steps, times, image, all of it. Where it does not, you get a partial import to tidy up by hand.

There is a bulk importer under the same menu that takes a list of URLs, which is the fastest way to migrate a bookmark folder.

<!-- screenshot: Mealie recipe view with ingredients and instructions -->

## 4. Set up the meal plan

**Meal Planner** assigns recipes to dates. The **Planner Rules** are the part worth configuring — they let you say "Fridays are takeaway" or "weekdays pick from the Quick tag", and then generate a week automatically.

Tag recipes as you import them. A library of 200 untagged recipes is a search problem; a tagged one is a meal planner.

## 5. Shopping lists

**Shopping Lists → Create**, then add a meal plan or individual recipes to it.

Mealie consolidates quantities — two recipes each wanting one onion produce one line for two onions. Configure **Food and Units** under settings so it recognises that "onion" and "onions" are the same thing.

The list is usable on a phone in the shop, which is the actual test of whether any of this was worth setting up.

## 6. Put it behind Traefik

```yaml
    environment:
      BASE_URL: https://mealie.example.com
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mealie.rule=Host(`mealie.example.com`)"
      - "traefik.http.routers.mealie.entrypoints=https"
      - "traefik.http.routers.mealie.tls.certresolver=letsencrypt"
      - "traefik.http.services.mealie.loadbalancer.server.port=9000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Note the service port is **9000**, the container's port, not the 9925 you published on the host.

`BASE_URL` must match the public address or share links and password reset emails point somewhere useless.

Mealie has its own user accounts, so it does not need [Tinyauth](tinyauth.md) in front — and should not have it, since the mobile experience relies on the app talking to the API.

## 7. Add users and households

**Settings → Users**. Households separate meal plans and shopping lists while sharing the recipe library, which is the right model for a shared house — everyone sees the same recipes, nobody's shopping list gets mixed up.

## Updating

```bash
cd /opt/docker/mealie
docker compose pull
docker compose up -d
```

Because the version is pinned, this only pulls what you asked for. To upgrade, edit the tag first and read the [release notes](https://github.com/mealie-recipes/mealie/releases) — upstream warns that manual steps are occasionally required, and the v1-to-v2 migration was a notable example.

Back up before any version bump.

## Backup

Mealie has a built-in backup under **Settings → Backups** that produces a downloadable archive, and that is the one to use for a restore into a different install.

For the host-level copy:

```bash
cd /opt/docker/mealie
docker compose stop
docker run --rm \
  -v mealie_mealie-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/mealie-$(date +%F).tar.gz -C /data .
docker compose start
```

Stop it first — SQLite.

## Troubleshooting

**Import produces an empty or partial recipe.** The site has no structured data. Nothing to fix; enter it manually, or find the recipe somewhere that publishes properly.

**"Database is locked" errors.** SQLite on network storage. See the warning at the top — move to local storage or Postgres.

**Container idles at high memory.** The `deploy.resources.limits.memory` block is missing.

**Images do not load behind the proxy.** `BASE_URL` is wrong or unset.

**Cannot log in after an update.** Check the logs for a failed migration. This is why you pin the version and back up first.

**Permission denied writing to the data directory.** PUID/PGID do not match the volume owner. On Unraid use 99/100.

**Recipe images vanished after a restore.** They live in the data volume alongside the database — restore the whole thing, not just the SQL.

## Where this sits in my lab

Mealie runs on the Docker VM on my first Proxmox node, with the data volume on local SSD rather than the array, specifically because of the SQLite warning above.

It replaced a bookmark folder of recipe sites and a shared notes app. The meal planner gets used more than expected; the recipe import is what made it stick, because migrating fifty bookmarked recipes took about ten minutes rather than an evening of copy-paste.
