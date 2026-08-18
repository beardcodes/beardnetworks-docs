# Tinyauth

Most of the services in this lab have their own login page, and a few have none at all. Tinyauth puts one authentication layer in front of all of them, at the reverse proxy, so a service with no auth of its own becomes something only you can reach.

It is the smallest way to solve that problem. Authelia and Authentik do more; Tinyauth does this one thing in a single container with no database.

[Tinyauth](https://tinyauth.app/) is an OpenID Certified™ authentication and authorization server designed to sit behind a reverse proxy.

1. **Forward auth** for Traefik, Caddy, nginx and others — the proxy asks Tinyauth before serving a request
2. **OAuth providers** — Google, GitHub, or any generic OIDC provider
3. **Simple user accounts** defined in an environment variable, no database to run
4. **TOTP** two-factor authentication
5. **Access control** by user, by domain, and by resource
6. **Genuinely small** — a single container, minimal memory

## How forward auth works

Worth understanding before you configure it, because the failure modes make no sense otherwise:

1. A request arrives at Traefik for `dockhand.home.lan`
2. Traefik pauses it and asks Tinyauth: "is this request authenticated?"
3. Tinyauth checks for a valid session cookie
4. **No** → redirect the browser to the Tinyauth login page
5. **Yes** → Tinyauth returns `200`, and Traefik forwards to the real service

The protected service never sees an unauthenticated request, and needs no modification. It also has no idea any of this happened, which is why forward auth works with applications that have no concept of users.

!!! note "Prerequisites"

    - [Traefik](traefik.md) running as your reverse proxy
    - A hostname for Tinyauth itself, e.g. `auth.example.com`
    - A hashed password — covered in step 1

## 1. Generate a password hash

Tinyauth stores users as `username:bcrypt-hash`. Generate one:

```bash
docker run --rm httpd:alpine htpasswd -nbB user "your-password"
```

Output looks like:

```
user:$2y$05$KX3Xk...
```

!!! danger "Dollar signs must be doubled inside a Compose file"

    Compose treats `$` as variable interpolation. A bcrypt hash is full of them, and pasting one raw produces a mangled hash and a login that silently never works.

    Every `$` becomes `$$`:

    ```
    $2y$05$KX3Xk...   ->   $$2y$$05$$KX3Xk...
    ```

    This is the single most common Tinyauth problem. If your password is "wrong" and you are certain it is right, this is why.

    Using an `.env` file instead avoids the escaping entirely, and is the better habit.

## 2. The Compose file

Upstream's example, showing Traefik, Tinyauth and a test service together:

```yaml
services:
  traefik:
    image: traefik:v3.6
    command: --api.insecure=true --providers.docker
    ports:
      - 80:80
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  whoami:
    image: traefik/whoami:latest
    labels:
      traefik.enable: true
      traefik.http.routers.whoami.rule: Host(`whoami.example.com`)
      traefik.http.routers.whoami.middlewares: tinyauth

  tinyauth:
    image: ghcr.io/tinyauthapp/tinyauth:v5
    environment:
      - TINYAUTH_APPURL=https://tinyauth.example.com
      - TINYAUTH_AUTH_USERS=user:$$2a$$10$$UdLYoJ5lgPsC0RKqYH/jMua7zIn0g9kPqWmhYayJYLaZQ/FTmH2/u # user:password
    volumes:
      - ./data:/data
    labels:
      traefik.enable: true
      traefik.http.routers.tinyauth.rule: Host(`tinyauth.example.com`)
      traefik.http.middlewares.tinyauth.forwardauth.address: http://tinyauth:3000/api/auth/traefik
```

Three lines are doing the real work:

1. **`TINYAUTH_APPURL`** — the public URL of Tinyauth itself. Redirects break if this is wrong, usually as a loop.
2. **`forwardauth.address`** — defines the middleware. Note it points at the *internal* container address and port 3000.
3. **`middlewares: tinyauth`** on `whoami` — this is what actually protects a service. The middleware existing does nothing until a router uses it.

For your own lab, adapt it to your existing Traefik rather than running a second one. Pin `:v5` — the major tag, not `latest`.

## 3. Adapt it to an existing Traefik setup

If Traefik is already running elsewhere, Tinyauth needs to join its network:

```yaml
services:
  tinyauth:
    image: ghcr.io/tinyauthapp/tinyauth:v5
    container_name: tinyauth
    restart: unless-stopped
    environment:
      - TINYAUTH_APPURL=https://auth.example.com
      - TINYAUTH_AUTH_USERS=${TINYAUTH_USERS}
      - TINYAUTH_SECURE_COOKIE=true
    volumes:
      - ./data:/data
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tinyauth.rule=Host(`auth.example.com`)"
      - "traefik.http.routers.tinyauth.entrypoints=https"
      - "traefik.http.routers.tinyauth.tls.certresolver=letsencrypt"
      - "traefik.http.services.tinyauth.loadbalancer.server.port=3000"
      - "traefik.http.middlewares.tinyauth.forwardauth.address=http://tinyauth:3000/api/auth/traefik"

networks:
  traefik-net:
    external: true
```

With the users in `.env`, where the `$` escaping problem does not exist:

```bash
TINYAUTH_USERS=user:$2y$05$KX3Xk...
```

```bash
chmod 600 .env
docker compose up -d
```

## 4. Protect a service

Add one label to anything you want behind the login:

```yaml
      - "traefik.http.routers.dockhand.middlewares=tinyauth@docker"
```

The `@docker` suffix matters — it tells Traefik the middleware was defined by the Docker provider. If Tinyauth is defined in a file instead, it is `tinyauth@file`.

Good candidates in this lab: [Dockhand](dockhand.md), [Uptime Kuma](uptime-kuma.md), [Prometheus](prometheus.md), and anything else whose own authentication is weak or absent.

!!! warning "Do not put forward auth in front of APIs"

    Tinyauth authenticates browsers, by redirecting them to a login page. An API client receiving a redirect to an HTML form will simply break.

    Do not protect: [Sonarr](sonarr.md) and [Radarr](radarr.md) if [Prowlarr](prowlarr.md) or [Seerr](seerr.md) talk to their APIs, any service another container calls programmatically, or webhook endpoints. Those need their own API-key auth, which they already have.

## 5. Turn on TOTP

Log in at `https://auth.example.com`, go to the account page, and enrol an authenticator app.

Given this one login now gates several admin interfaces, second factor is worth the thirty seconds.

## 6. OAuth, optionally

To sign in with Google or GitHub instead of a local password:

```yaml
      - TINYAUTH_GITHUB_CLIENT_ID=${GITHUB_CLIENT_ID}
      - TINYAUTH_GITHUB_CLIENT_SECRET=${GITHUB_CLIENT_SECRET}
      - TINYAUTH_OAUTH_WHITELIST=me@example.com
```

**Set the whitelist.** Without it, any GitHub account in the world can authenticate — the OAuth provider confirms *who* someone is, not whether they are allowed in. That distinction has caught out plenty of people.

## Updating

```bash
cd /opt/docker/tinyauth
docker compose pull
docker compose up -d
```

Pinning `:v5` keeps you on that major version. Check the [release notes](https://github.com/tinyauthapp/tinyauth/releases) before moving major versions — configuration keys have changed between them.

## Backup

```bash
sudo tar czf /mnt/user/backups/tinyauth-$(date +%F).tar.gz -C /opt/docker/tinyauth data .env
```

The `.env` holds your password hashes and OAuth secrets. Treat the archive accordingly.

## Troubleshooting

**Password rejected even though it is correct.** The `$$` escaping, or a hash copied with a trailing newline. Move the users into `.env` and try again.

**Infinite redirect loop.** `TINYAUTH_APPURL` does not match the URL in the browser. It must be the exact public address, scheme included.

**Login succeeds, then the service still asks again.** Cookie domain. Tinyauth and the protected service need a shared parent domain — `auth.example.com` protecting `app.example.com` works; protecting a completely different domain does not.

**404 on the Tinyauth hostname.** The router is not matching. Check the rule and that the container is on the Traefik network.

**Every request returns 500.** Traefik cannot reach Tinyauth. The `forwardauth.address` must be the internal container name and port 3000, not the public URL.

**An app broke after adding the middleware.** It is probably an API client. See the warning in step 4.

**Cookie not set on plain HTTP.** `TINYAUTH_SECURE_COOKIE=true` requires HTTPS. Either use TLS, which you should, or turn it off for local testing only.

## Where this sits in my lab

Tinyauth runs on `proxmox2` next to [Traefik](traefik.md), protecting the admin interfaces that either have weak logins or none — Dockhand, Uptime Kuma, Prometheus.

It deliberately does *not* sit in front of the *arr stack, because those talk to each other over APIs and forward auth would break the whole chain. Those stay LAN-only and reached through [wg-easy](wg-easy.md) instead — a VPN and a forward-auth proxy solve overlapping problems, and knowing which one applies where is most of the work.
