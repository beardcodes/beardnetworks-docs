# Dockhand

[Portainer](portainer.md) has been the default Docker UI for years, and it shows — the interface carries a lot of history and a lot of Swarm and Kubernetes machinery most homelabs never touch. Dockhand is a newer take: the same core job, a modern interface, and a Compose editor that is genuinely pleasant to use.

[Dockhand](https://dockhand.pro) is a Docker management UI built on SvelteKit and Bun, running on a from-scratch Wolfi base image.

1. **Container management** — start, stop, restart, with live status, resource use and port mappings
2. **Compose stacks** with a visual YAML editor, environment variables side by side
3. **Git integration** — deploy stacks from a repository, with webhooks and auto-sync
4. **Multi-environment** — manage local and remote Docker hosts from one place
5. **Terminal and log streaming**, plus a file browser into running containers
6. **OIDC SSO** and local users, with RBAC in the Enterprise tier

## Dockhand or Portainer?

Both mount the Docker socket and show you containers. The differences that matter:

| | Dockhand | Portainer CE |
|---|---|---|
| Age and ecosystem | New, smaller community | Years of guides and StackOverflow answers |
| Interface | Modern, quick | Functional, dated in places |
| Compose editing | Strong — side-by-side YAML and env | Workable |
| Git-backed stacks | Yes, with webhooks | Yes, with polling |
| Kubernetes | No | Yes |
| Docker Swarm | No | Yes |
| File browser into containers | Yes | No |

If you run Kubernetes or Swarm, Portainer covers ground Dockhand does not. For a Compose-only homelab, Dockhand is the nicer daily driver — and there is no reason not to run both against the same socket while you decide.

!!! warning "Anything with the Docker socket has root on the host"

    Dockhand mounts `/var/run/docker.sock`. A process that can talk to the socket can start a privileged container and take over the machine. That is a property of the socket, not a flaw in Dockhand — but it means this UI never faces the internet, and it gets a real password.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - Port 3000 free, or a willingness to remap it

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/dockhand && cd /opt/docker/dockhand
sudo nano docker-compose.yml
```

Upstream's file:

```yaml
services:
  dockhand:
    image: fnsys/dockhand:latest
    container_name: dockhand
    restart: unless-stopped
    ports:
      - 3000:3000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - dockhand_data:/app/data

volumes:
  dockhand_data:
```

!!! note "Port 3000 is crowded"

    [Grafana](grafana.md), [Karakeep](karakeep.md) and Dockhand all want 3000. On a host running more than one, remap the host side — `3010:3000` — and leave the container side alone.

```bash
docker compose up -d
docker compose logs -f
```

## 2. Create the admin account

Browse to `http://<host-ip>:3000`. First run walks you through creating the initial administrator.

Use a real password. This account can deploy arbitrary containers on your host.

## 3. Look around

The local Docker environment is picked up automatically from the mounted socket. You should immediately see every container on the host, with live CPU and memory.

Worth knowing where things are:

- **Environments** — Docker hosts. One to start with; add more in step 5.
- **Containers** — status, ports, resources, plus the log stream and shell
- **Stacks** — Compose deployments, and where the editor lives
- **Images** — tags, sizes, update availability, and cleanup

<!-- screenshot: Dockhand environment dashboard with live host metrics -->

## 4. Deploy a stack

**Stacks → New Stack** gives you the YAML editor. Paste a Compose file, set environment variables in the adjacent pane, deploy.

The editor is the reason to use Dockhand over the alternatives — env vars sit next to the YAML that consumes them, so `${THING}` is not a guess.

!!! note "Stacks created on the CLI stay on the CLI"

    Like Portainer, Dockhand can only manage stacks it has the Compose file for. Containers started with `docker compose up` from a shell appear under Containers, but not as an editable stack.

    Pick one place per app and stay there. Two competing definitions of the same stack is how you lose an evening.

## 5. Add a second Docker host

**Environments → Add Environment** connects a remote host, so one UI covers the whole lab.

Connecting over TCP means exposing the Docker API, which is the socket problem again but over the network. Do not enable plain `tcp://` on 2375. Either use an SSH connection, or mTLS on 2376 with certificates.

If you already run the [Portainer agent](portainer.md#5-add-a-second-docker-host) on those hosts, that is a separate mechanism — Dockhand does not use it.

## 6. Git-backed stacks

**Stacks → New → From Git** points at a repository with a Compose file. With a webhook configured, a push redeploys.

Combined with a [Gitea](https://about.gitea.com/) instance, that makes the repo the source of truth and the UI a viewer — which is the arrangement that survives contact with a lab you have not touched in six months. You can read what is deployed without logging into anything.

## 7. Put it behind Traefik

```yaml
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dockhand.rule=Host(`dockhand.home.lan`)"
      - "traefik.http.routers.dockhand.entrypoints=https"
      - "traefik.http.routers.dockhand.tls.certresolver=letsencrypt"
      - "traefik.http.services.dockhand.loadbalancer.server.port=3000"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

Drop the `ports:` block when you do this. Consider adding [Tinyauth](tinyauth.md) as a second factor in front — for a socket-mounting admin UI, a single password is thin.

## Updating

```bash
cd /opt/docker/dockhand
docker compose pull
docker compose up -d
```

Dockhand is young and moves quickly. Read the release notes before upgrading, and take the backup below first — this is not a project with years of upgrade-path scar tissue behind it yet.

## Backup

```bash
docker compose stop
docker run --rm \
  -v dockhand_dockhand_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/dockhand-$(date +%F).tar.gz -C /data .
docker compose start
```

The volume holds the database — users, environments and stack definitions. Git-backed stacks are reproducible from the repo; anything you pasted into the editor is not.

## Troubleshooting

**Blank dashboard, or no containers listed.** The socket mount. `docker exec dockhand ls -l /var/run/docker.sock`.

**Port 3000 already allocated.** Something else has it. `ss -tlnp | grep 3000`, then remap the host side.

**Remote environment will not connect.** Check the Docker API is actually reachable and, if using TLS, that the certificates match the hostname you connected to.

**Stack deploys but the container immediately exits.** Read the container log, not the stack log — the stack succeeded in creating something that then failed on its own terms.

**Cannot log in after an update.** Check the logs for a migration error. Restore the backup if the database did not migrate cleanly.

## Where this sits in my lab

Dockhand runs on the Docker VM on my first Proxmox node, alongside [Portainer](portainer.md) — both pointed at the same socket while I work out which one I actually reach for.

Early verdict: Dockhand for anything involving editing a Compose file, Portainer for the multi-host view, since its agent is already deployed everywhere. Neither is exposed beyond the LAN.
