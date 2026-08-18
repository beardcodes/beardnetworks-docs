# Portainer

Managing Docker from the CLI is fine until you are three SSH sessions deep trying to remember which host runs which stack. Portainer puts a web UI on top of the Docker socket: containers, images, volumes, networks, logs and a shell, in a browser, across multiple hosts.

This guide sets up Portainer CE, connects a second Docker host as an agent, and puts the UI behind Traefik so you stop typing port numbers.

[Portainer](https://www.portainer.io/) is a management UI for container platforms — Docker, Docker Swarm and Kubernetes. The Community Edition is free and covers everything a homelab needs.

What it actually gives you:

1. Container lifecycle — start, stop, recreate, inspect, without remembering flags
2. Live logs and an in-browser terminal, so you can `exec` into a container from your phone
3. **Stacks** — deploy and edit Compose files from the UI, including straight from a Git repo
4. Multi-endpoint management — one UI for every Docker host you own
5. Image, volume and network management with a working "what is safe to prune" view
6. Users, teams and RBAC if more than one person touches the lab

!!! warning "Portainer has root on your host"

    Portainer works by mounting `/var/run/docker.sock`. Anything that can talk to the Docker socket can start a privileged container and own the host. That is not a Portainer flaw, it is what the socket is — but it means you should never expose the Portainer UI to the internet, and you should give it a real password.

!!! note "Prerequisites"

    - A Linux host with Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)
    - Port 9443 free
    - Optional: [Traefik](traefik.md) already running for the reverse proxy step

## 1. Create the data volume

Portainer keeps its database, users and endpoint config in `/data`. That has to be a named volume or you will lose your setup on the first container recreate.

```bash
docker volume create portainer_data
```

## 2. Write the Compose file

Make a directory for the stack. I keep everything under `/opt/docker/<app>` so backups are one predictable path:

```bash
sudo mkdir -p /opt/docker/portainer && cd /opt/docker/portainer
sudo nano docker-compose.yml
```

```yaml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:lts
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    ports:
      - 9443:9443
      - 8000:8000

volumes:
  portainer_data:
    name: portainer_data
```

Breaking down the parts that matter:

1. `portainer/portainer-ce:lts` — the LTS tag, not `latest`. Portainer ships breaking UI changes on the current branch; LTS gets security fixes without moving your cheese.
2. `/var/run/docker.sock:/var/run/docker.sock` — the whole point. This is how Portainer sees your containers.
3. `portainer_data:/data` — users, settings and endpoint definitions. Back this up.
4. `9443` — the HTTPS UI. Portainer generates its own self-signed cert on first boot.
5. `8000` — the Edge agent tunnel. If you are not using Edge agents, delete this line.
6. `restart: always` — Portainer should come back before the things it manages.

## 3. Start it

```bash
docker compose up -d
docker compose logs -f
```

## 4. Create the admin user

Open `https://<host-ip>:9443` and accept the self-signed certificate warning.

You get an admin setup screen. **Do this within a few minutes of starting the container** — Portainer times out the initial setup window as a security measure, and if you miss it you have to restart the container to get the form back. Password minimum is 12 characters.

On the next screen, choose **Get Started** to manage the local Docker environment. The **Environments → local** view should now list every container on the host.

<!-- screenshot: Portainer dashboard showing the local endpoint container count -->

## 5. Add a second Docker host

The Portainer Agent exposes a remote host's Docker socket to your existing UI. On the **second** machine:

```yaml
services:
  agent:
    container_name: portainer_agent
    image: portainer/agent:lts
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    ports:
      - 9001:9001
```

```bash
docker compose up -d
```

Then in the Portainer UI: **Environments → Add environment → Docker Standalone → Agent**, and give it `<second-host-ip>:9001`.

!!! danger "Port 9001 is unauthenticated by default"

    The agent trusts anything that can reach it. Keep 9001 on your LAN or a WireGuard subnet — never forward it at the router, and firewall it to the Portainer host's IP:

    ```bash
    sudo ufw allow from 192.168.1.10 to any port 9001 proto tcp
    ```

## 6. Put it behind Traefik

Reaching for `:9443` gets old. Add Portainer to your Traefik network and give it a hostname.

Note the `scheme=https` line — Portainer speaks HTTPS on 9443 with a self-signed cert, so Traefik has to be told to talk HTTPS to the backend *and* not to validate that cert. This is the step most guides get wrong.

```yaml
services:
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:lts
    restart: always
    security_opt:
      - no-new-privileges:true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.example.com`)"
      - "traefik.http.routers.portainer.entrypoints=https"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9443"
      - "traefik.http.services.portainer.loadbalancer.server.scheme=https"
      - "traefik.http.services.portainer.loadbalancer.serverstransport=insecure@file"

volumes:
  portainer_data:
    name: portainer_data

networks:
  traefik-net:
    external: true
```

That last label needs a matching entry in your Traefik dynamic config file:

```yaml
http:
  serversTransports:
    insecure:
      insecureSkipVerify: true
```

The `ports:` block is gone — traffic now arrives through Traefik only. Recreate:

```bash
docker compose up -d --force-recreate
```

## 7. Deploy your first stack from the UI

**Stacks → Add stack** gives you three ways in: paste a Compose file into the web editor, upload one, or point at a Git repository.

The Git option is the interesting one. Give it a repo URL and a path to `docker-compose.yml`, enable **GitOps updates** with a 5-minute polling interval, and Portainer will redeploy the stack whenever you push. That turns your Gitea instance into the source of truth for the lab, and the UI into a viewer rather than a place where undocumented changes happen.

!!! note "Stacks created outside Portainer show as 'limited'"

    Containers started by `docker compose up` on the CLI appear under **Containers**, but Portainer will not let you edit them as a stack because it has no copy of the Compose file. Either move the file into a Portainer stack, or accept that CLI stacks stay CLI stacks. Mixing the two on the same app is how you end up with two competing definitions.

## Updating

Portainer will not update itself — the "update" it offers in the UI applies to your containers, not to Portainer.

```bash
cd /opt/docker/portainer
docker compose pull
docker compose up -d
```

Your data lives in the `portainer_data` volume, so this is safe. If you run [Diun](diun.md), it will tell you when a new LTS image lands.

## Backup

Two options, and you want both:

**From the UI:** **Settings → Backup Portainer** produces an encrypted archive of the whole config. There is a scheduled S3 option in CE too.

**From the host:**

```bash
docker run --rm \
  -v portainer_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/portainer-$(date +%F).tar.gz -C /data .
```

## Troubleshooting

**"Failed to load resource" or a blank dashboard.** Almost always the Docker socket. Check the container can read it: `docker exec portainer ls -l /var/run/docker.sock`.

**Locked out of the setup screen.** Restart the container: `docker restart portainer`, then complete the form promptly.

**Forgotten admin password.** Stop Portainer, then run the password reset helper against the same volume:

```bash
docker stop portainer
docker run --rm -v portainer_data:/data portainer/helper-reset-password
docker start portainer
```

It prints a new random password to the terminal.

**502 Bad Gateway through Traefik.** You missed the `scheme=https` label, or the `serversTransports` entry. Traefik is speaking plain HTTP to an HTTPS port.

**Agent endpoint shows as down.** From the Portainer host: `curl -k https://<agent-ip>:9001/ping`. If that fails it is firewall or routing, not Portainer.

## Where this sits in my lab

Portainer runs on the Docker VM on my first Proxmox node, with agents on the Unraid box and on `proxmox2`. That gives one URL for all three hosts — the *arr stack, the networking services and the utilities — without three sets of SSH keys and three terminal tabs.

I still write Compose files in an editor and keep them in Gitea. Portainer is where I look at logs, restart something at 1am from my phone, and check what is eating disk. Treating it as a viewer rather than an editor keeps the Git repo honest.
