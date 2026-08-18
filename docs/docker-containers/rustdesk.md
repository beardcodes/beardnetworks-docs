# RustDesk

TeamViewer works until it decides your personal use looks commercial and locks you out mid-session. RustDesk is the open-source replacement, and self-hosting the server means the connection never touches anyone else's infrastructure.

The client is free either way — what you are deploying here is the rendezvous and relay server that lets two machines find each other.

[RustDesk](https://rustdesk.com/) is an open-source remote desktop application. `rustdesk-server` is the self-hosted signalling and relay component.

1. **Cross-platform** — Windows, macOS, Linux, Android, iOS, and a web client
2. **Direct connections** where the network allows, falling back to relay
3. **End-to-end encrypted**, with a key pair generated on your server
4. **File transfer** and clipboard sync
5. **Unattended access** with a permanent password
6. **No account required** and no per-seat licensing

## Two containers, and what they do

This confuses people, so it is worth stating plainly:

| Container | Role | Ports |
|---|---|---|
| **hbbs** | Rendezvous / signalling — how clients find each other | 21115, 21116 (TCP+UDP), 21118 |
| **hbbr** | Relay — carries traffic when a direct connection fails | 21117, 21119 |

Clients try to connect directly, peer to peer. When NAT prevents that, traffic goes through `hbbr`. On a home connection with two machines behind different networks, the relay is used more often than you would expect — so do not skip it.

The `21118` and `21119` ports are for the web client. Omit them if you do not need browser access.

!!! note "Prerequisites"

    - A host reachable from wherever you will connect *from* — a VPS, or your home server with ports forwarded
    - A static public IP or a DDNS hostname
    - The ability to forward five ports

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/rustdesk && cd /opt/docker/rustdesk
sudo nano docker-compose.yml
```

Upstream's stack, with the relay hostname set:

```yaml
networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    command: hbbs -r rustdesk.example.com:21117
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    command: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
```

**`-r rustdesk.example.com:21117`** tells `hbbs` where to send clients for relaying. Set it to your public hostname or IP. Get this wrong and direct connections work while relayed ones fail — which presents as "it works at home but not from the office".

Both containers share `./data` so they use the same key pair.

```bash
docker compose up -d
docker compose logs -f
```

## 2. Forward the ports

At your router, forward to the host:

| Port | Protocol |
|---|---|
| 21115 | TCP |
| 21116 | **TCP and UDP** |
| 21117 | TCP |
| 21118 | TCP (web client only) |
| 21119 | TCP (web client only) |

21116 needs **both** TCP and UDP — UDP is the heartbeat that keeps clients registered. Forwarding only TCP produces clients that appear online briefly then go offline.

!!! warning "This exposes a service to the internet"

    Unlike most things in this documentation, RustDesk's server has to be publicly reachable to be useful. Keep it updated, and consider running it on a cheap VPS rather than your home connection — that way a compromise lands somewhere that is not holding your media library.

    If everyone connecting is already on your VPN, you do not need RustDesk at all — use [wg-easy](wg-easy.md) plus [Nexterm](nexterm.md) or plain RDP instead.

## 3. Get the public key

On first start, the server generates a key pair in `./data`:

```bash
cat /opt/docker/rustdesk/data/id_ed25519.pub
```

That string is your public key. Every client needs it, along with the server address.

!!! danger "Back up the key pair now"

    Losing `id_ed25519` means every client must be reconfigured with the new key. Copy both files somewhere safe before you deploy to a dozen machines.

    Conversely, the key is what stops arbitrary clients using your relay — treat the private key as a secret.

## 4. Configure a client

In the RustDesk client: **⋮ → Network → ID/Relay Server**

| Field | Value |
|---|---|
| ID Server | `rustdesk.example.com` |
| Relay Server | `rustdesk.example.com` |
| API Server | leave blank |
| Key | the contents of `id_ed25519.pub` |

The status indicator at the bottom left should go green and read "Ready".

<!-- screenshot: RustDesk client network settings with custom server configured -->

For deploying to many machines, RustDesk supports encoding the whole configuration into the client filename or pushing it via a config string — worth looking up if you are setting up more than three or four.

## 5. Set unattended access

On any machine you want to reach without someone clicking Accept, set a **permanent password** in the client under **Security**.

Use a strong one. This is a full remote desktop session on that machine, and the ID is a short number that is not hard to guess.

Also worth setting under **Security → Permissions**: disable file transfer or clipboard if you do not need them.

## 6. The web client, optionally

Ports 21118 and 21119 serve the browser client. Useful when you are on a locked-down machine where you cannot install software.

It is the weakest part of the project — expect worse performance than the native client, and treat it as a fallback.

## Updating

```bash
cd /opt/docker/rustdesk
docker compose pull
docker compose up -d
```

Server and client versions should stay roughly in step. A very old server with new clients occasionally produces connection failures that look like network problems.

## Backup

Small but critical:

```bash
sudo tar czf /mnt/user/backups/rustdesk-$(date +%F).tar.gz -C /opt/docker/rustdesk data
```

`data` holds the key pair and the ID database. The keys are the part you cannot regenerate without reconfiguring every client.

## Troubleshooting

**Client shows "Ready" but connections time out.** Relay is not reachable. Check 21117 is forwarded and that `-r` names the correct public host.

**Clients go offline after a few seconds.** UDP 21116 is not forwarded. This is the most common RustDesk problem.

**"Key mismatch" or connection refused.** The key in the client does not match `id_ed25519.pub`. Re-copy it — a truncated paste is easy.

**Works on the LAN, not from outside.** Port forwarding, or your ISP uses CGNAT. A `100.64.x.x` hop in `traceroute` means inbound connections are impossible; use a VPS or a VPN instead.

**Very slow sessions.** Everything is relaying rather than connecting directly. Check the connection type in the client's status. Relayed sessions are bounded by your home upload speed.

**hbbs restarts repeatedly.** Usually a permissions problem on `./data`, or `hbbr` not yet up. Check `docker compose logs hbbs`.

## Where this sits in my lab

RustDesk's server runs on `proxmox2` with the five ports forwarded — one of only three things in this lab reachable from the internet, alongside [Seerr](seerr.md) and [Vaultwarden](vaultwarden.md).

It is what I use to help family with their machines, which is the actual use case: they install one client, I send them a config, and I stop having to explain things over the phone. For my own machines I use [wg-easy](wg-easy.md) and SSH, because a VPN plus a terminal is faster than any remote desktop.
