# Excalidraw

Excalidraw is a whiteboard that draws everything in a hand-sketched style, which sounds like a gimmick and turns out to be the point: a diagram that looks deliberately rough invites people to argue with it. A polished one looks finished.

It is the fastest way to sketch a network topology, a container layout, or the thing you are about to explain badly in text.

[Excalidraw](https://excalidraw.com/) is a virtual whiteboard for sketching hand-drawn-like diagrams.

1. **Hand-drawn aesthetic** that keeps diagrams feeling provisional
2. **Runs entirely in the browser** — the container serves static files
3. **Libraries** of ready-made shapes, including cloud and infrastructure icon sets
4. **Export** to PNG, SVG, or a `.excalidraw` JSON file
5. **Embeddable images and text**, with a diagram-as-code mode via Mermaid
6. **Optional live collaboration** with a second container

!!! danger "The self-hosted version does not store your drawings"

    This is the thing to understand before you deploy it, because it surprises everyone.

    The container is a **static web server**. Your drawings live in your browser's local storage — not on the server. That means:

    - Clearing browser data deletes your work
    - A drawing made on your laptop does not appear on your phone
    - There is nothing on the server to back up, because there is nothing on the server

    **Export anything you want to keep.** The `.excalidraw` file is JSON and reopens perfectly. Treat this as a sketchpad, not a document store — for persistent, synced drawings you want [Excalidraw+](https://plus.excalidraw.com/) or a different tool entirely.

!!! note "Prerequisites"

    - Docker and the Compose plugin — see the [Docker guide](../host-setup/docker.md)

## 1. The Compose file

!!! warning "Do not use the compose file in the repository"

    `docker-compose.yml` in the Excalidraw repository is a **development** setup — it builds from source, mounts the working tree, and sets `NODE_ENV=development`. It is for people hacking on Excalidraw.

    For self-hosting, use the published image.

```bash
sudo mkdir -p /opt/docker/excalidraw && cd /opt/docker/excalidraw
sudo nano docker-compose.yml
```

```yaml
services:
  excalidraw:
    image: excalidraw/excalidraw:latest
    container_name: excalidraw
    restart: unless-stopped
    ports:
      - 8083:80
```

No volumes, because there is no state. Port 80 inside; 8083 on the host to stay clear of the 8080 crowd.

```bash
docker compose up -d
```

Browse to `http://<host-ip>:8083`.

## 2. Add live collaboration, optionally

Real-time collaboration needs a second container, `excalidraw-room`, which relays encrypted updates between participants:

```yaml
services:
  excalidraw:
    image: excalidraw/excalidraw:latest
    container_name: excalidraw
    restart: unless-stopped
    ports:
      - 8083:80
    environment:
      - VITE_APP_WS_SERVER_URL=https://excalidraw-room.example.com

  excalidraw-room:
    image: excalidraw/excalidraw-room:latest
    container_name: excalidraw-room
    restart: unless-stopped
    ports:
      - 8084:80
```

Two caveats worth knowing before you spend an evening on this:

1. **`VITE_*` variables are build-time**, not runtime. The published image was built with defaults baked in, so setting this at runtime may not take effect — you may need to build your own image with the variable set. Check the current documentation before assuming it works.
2. **The room server needs WebSockets** through your reverse proxy. In [NPM](nginx-proxy-manager.md) that is the Websockets Support toggle; Traefik handles it by default.

For a single-user homelab sketchpad, skip this entirely.

## 3. Put it behind Traefik

```yaml
services:
  excalidraw:
    image: excalidraw/excalidraw:latest
    container_name: excalidraw
    restart: unless-stopped
    networks:
      - traefik-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.excalidraw.rule=Host(`draw.home.lan`)"
      - "traefik.http.routers.excalidraw.entrypoints=https"
      - "traefik.http.routers.excalidraw.tls.certresolver=letsencrypt"
      - "traefik.http.services.excalidraw.loadbalancer.server.port=80"
      - "traefik.docker.network=traefik-net"

networks:
  traefik-net:
    external: true
```

!!! warning "Local storage is per-origin"

    Because drawings live in browser storage keyed to the origin, changing how you reach Excalidraw changes where your drawings appear to be.

    Access it at `http://192.168.1.20:8083` and later at `https://draw.home.lan`, and the second one will look empty — your work is still under the first origin, not lost, but not visible either.

    Pick the hostname you intend to keep **before** you start drawing on it.

No authentication needed — there is nothing stored server-side to protect.

## 4. Use it for diagrams that belong in your docs

Excalidraw exports SVG, which is what you want for documentation: it scales, it stays sharp, and the file is text.

Better still, an exported PNG or SVG from Excalidraw can be **re-imported** — the scene data is embedded in the file. So the exported image *is* the source file, and you do not have to keep two copies in sync. Tick "Embed scene" on export.

Its Mermaid import is also useful: write a flowchart as text, convert it to Excalidraw shapes, then adjust by hand. Faster than drawing boxes, more flexible than pure Mermaid.

<!-- screenshot: Excalidraw canvas with a homelab network diagram -->

## 5. Add a shape library

**Library → Browse libraries** opens the public collection: AWS, Azure, Kubernetes, network equipment, UI wireframe kits.

Libraries are stored in your browser, same as drawings — so they follow the same rules about origins and clearing data.

## Updating

```bash
cd /opt/docker/excalidraw
docker compose pull
docker compose up -d
```

Safe: no state, no migrations. Hard-refresh the browser afterwards, as the assets cache aggressively.

Your drawings are unaffected by updates because they were never on the server in the first place.

## Backup

Nothing on the server to back up.

Your drawings are in browser local storage. To keep any of them, **File → Save to disk**, and store the `.excalidraw` files somewhere real — a Git repo alongside the documentation they illustrate is the natural home.

Keep the Compose file in Git and the service is reproducible in seconds.

## Troubleshooting

**All my drawings vanished.** Either browser data was cleared, or you are reaching the site at a different origin than before. Check the old URL — the work is usually still there.

**Collaboration button does nothing.** No room server configured, or the `VITE_APP_WS_SERVER_URL` build-time issue in step 2.

**Collaboration connects then drops.** WebSockets not passing through the reverse proxy.

**Blank page.** Check you are proxying to port 80, and hard-refresh.

**Export to PNG produces a blank image.** Usually a very large canvas. Select the elements you want and export the selection instead.

**Fonts look wrong on an exported SVG.** The hand-drawn font is not embedded by default; export as PNG, or embed fonts if the option is available in your version.

## Where this sits in my lab

Excalidraw runs on the Docker VM on my first Proxmox node at a fixed internal hostname — fixed specifically because of the local-storage-per-origin problem above, which I discovered the way everyone does.

It gets used for the diagrams that end up explaining this lab to myself six months later: what talks to what, which host runs which stack, how traffic reaches a container. Exported as SVG with the scene embedded, stored next to the docs in Gitea, so the diagram and the file that produced it are the same object.
