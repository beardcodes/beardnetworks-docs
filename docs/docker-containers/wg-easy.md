# wg-easy

WireGuard is the right way to get back into your homelab from outside it. The only awkward part is key management, which is what wg-easy solves: a web UI that generates clients, shows you a QR code, and gets a phone connected in about fifteen seconds.

This is the single most useful thing you can put on a homelab, because it means nothing else needs to face the internet.

!!! warning "Version 15 changed everything"

    wg-easy v15 is a rewrite. The `WG_HOST` and `PASSWORD` environment variables are **gone**, replaced by a web-based setup wizard. It also now wants a dedicated Docker network with static IPs. Guides written for v13 or v14 will not work, and upgrading in place from v14 requires a migration. This guide covers v15.

[wg-easy](https://github.com/wg-easy/wg-easy) is WireGuard plus a web UI, in one container.

1. **Client management in a browser** — add, remove, enable and disable peers without touching a config file
2. **QR codes** for mobile clients, and downloadable `.conf` files for desktop
3. **Live traffic stats** per client, so you can see who is actually connected
4. **Configurable DNS** per client — point them at your AdGuard for ad-blocking on mobile data
5. **IPv6 support**, on by default in v15
6. **One UDP port** to forward, and nothing else exposed

## Why this beats exposing services directly

Every service you port-forward is a service you have to keep patched, monitor for brute force, and worry about. A VPN collapses that into one attack surface: a single UDP port running a protocol that does not respond at all to unauthenticated packets.

An unauthenticated port scan of your WireGuard port returns nothing. It looks closed. That property is why WireGuard is the correct default for remote access, and why most of the rest of this lab is deliberately LAN-only.

!!! note "Prerequisites"

    - A host with Docker — see the [Docker guide](../host-setup/docker.md)
    - The ability to forward **UDP 51820** at your router
    - A static public IP, or a dynamic DNS hostname
    - A LAN with a subnet that is not `192.168.0.0/24` or `192.168.1.0/24` if you can help it — see the warning in step 6

## 1. The Compose file

```bash
sudo mkdir -p /opt/docker/wg-easy && cd /opt/docker/wg-easy
sudo nano docker-compose.yml
```

This is upstream's v15 configuration:

```yaml
volumes:
  etc_wireguard:

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
```

What the unusual parts do:

1. `NET_ADMIN` and `SYS_MODULE` — required to create the WireGuard interface and load the kernel module. This container genuinely needs them.
2. `/lib/modules:ro` — lets it load the host's `wireguard` module on kernels that do not have it built in.
3. The `sysctls` block — enables IP forwarding inside the container, which is what makes it a gateway rather than an endpoint.
4. **51820/udp** is WireGuard itself. **51821/tcp** is the web UI.
5. The dedicated `wg` network with static IPs is new in v15 and not optional.

```bash
docker compose up -d
docker compose logs -f
```

## 2. Forward the port

At your router, forward **UDP 51820** to the host running this container. Only that one port. The web UI on 51821 should stay on the LAN.

If you do not have a static public IP, set up dynamic DNS now — most routers have a built-in client for DuckDNS or similar. You will need a hostname in step 3.

## 3. Run the setup wizard

Browse to `http://<host-ip>:51821`.

v15 gives you a setup wizard instead of environment variables:

1. **Create the admin account** — username and a strong password. This replaces the old `PASSWORD` env var.
2. **Host** — the public hostname or IP clients will connect to, e.g. `vpn.example.com` or your DDNS name. Get this right; it is baked into every client config you generate afterwards.
3. **Port** — 51820, matching your forward.

The wizard writes to the `etc_wireguard` volume, so this is a one-time thing.

<!-- screenshot: wg-easy setup wizard host and port configuration -->

## 4. Add a client

**New Client**, give it a name — use the device name, not the person's, since most people have three.

You get a QR code and a downloadable `.conf`. On mobile, install the official WireGuard app and scan the code. On desktop, import the file.

Connect, then check `https://ipinfo.io` on the client — it should show your home IP, not the phone network's.

## 5. Set the client DNS

**Admin → Configuration**, set the DNS server clients should use.

Point it at your [AdGuard](adguard.md) instance — for example `192.168.1.5`. Two things follow:

- Ad-blocking works on mobile data, anywhere in the world
- Internal hostnames resolve, so `karakeep.home.lan` works over the VPN exactly as it does at home

That second point is what makes the VPN genuinely pleasant to use rather than a thing you tolerate.

## 6. Choose: full tunnel or split tunnel

**Admin → Configuration → Allowed IPs** controls what traffic goes through the tunnel.

| Setting | Behaviour | Use when |
|---|---|---|
| `0.0.0.0/0, ::/0` | Everything routes through home | On untrusted Wi-Fi, or you want your home ad-blocking and IP everywhere |
| `192.168.1.0/24` | Only homelab traffic | Day-to-day; leaves normal browsing on the local connection |

Full tunnel is the default and is the safer choice on hotel and airport Wi-Fi.

!!! danger "Subnet collisions will ruin your day"

    If your home LAN is `192.168.1.0/24` and the coffee shop you are sitting in also uses `192.168.1.0/24`, split-tunnel routing becomes ambiguous and your connection to home silently fails — or worse, you reach *their* device at `192.168.1.5`.

    Use an unusual subnet at home: `10.13.37.0/24`, `192.168.88.0/24`, anything that is not the default two. This is worth renumbering your LAN for if you travel.

## 7. Reach the UI safely

Port 51821 should not be published to the internet. Two reasonable approaches:

**LAN and VPN only** — leave it as-is. You administer wg-easy from home, or from a device already connected to the VPN. This is the simplest and most defensible option.

**Behind Traefik** with authentication, if you must reach it remotely:

```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wg-easy.rule=Host(`vpn-admin.example.com`)"
      - "traefik.http.routers.wg-easy.entrypoints=https"
      - "traefik.http.routers.wg-easy.tls.certresolver=letsencrypt"
      - "traefik.http.services.wg-easy.loadbalancer.server.port=51821"
      - "traefik.http.routers.wg-easy.middlewares=authelia@docker"
```

The chicken-and-egg problem is real: if the VPN is down, you cannot use the VPN to fix the VPN. Keep SSH access to the host as your fallback.

## Updating

```bash
cd /opt/docker/wg-easy
docker compose pull
docker compose up -d
```

Pin the major version (`:15`) rather than using `:latest`. As the v14 to v15 jump demonstrates, this project makes breaking changes at major versions, and having that happen unattended while you are away from home is exactly the wrong time to find out.

## Backup

The `etc_wireguard` volume holds the server keys and every client config. Lose it and every client has to be re-enrolled.

```bash
docker run --rm \
  -v wg-easy_etc_wireguard:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/wg-easy-$(date +%F).tar.gz -C /data .
```

Treat this archive as a credential. It contains private keys that grant access to your network.

## Troubleshooting

**Handshake never completes.** The port forward. Check it from outside your network — `nc -zvu <public-ip> 51820` from a phone on mobile data. Also confirm your ISP is not using CGNAT, which makes inbound connections impossible regardless of your router config. If `traceroute` shows a `100.64.x.x` hop, you are behind CGNAT and need a different approach entirely.

**Connects but no traffic passes.** IP forwarding. Confirm the `sysctls` block is present, and that the container has `NET_ADMIN`.

**Connects, can ping the server, cannot reach other LAN devices.** Allowed IPs on the client does not include your LAN subnet, or your router lacks a return route to the VPN subnet.

**"Unable to load kernel module wireguard".** The `/lib/modules` mount is missing, or the host kernel predates 5.6. Install `wireguard-dkms` on the host.

**Web UI unreachable after upgrading from v14.** Expected — v15 changed the config format. Check the logs; it will tell you if it needs a migration. This is why you pin the major version.

**DNS does not resolve over the tunnel.** The DNS server you configured is not reachable from the WireGuard subnet. Check AdGuard is listening on all interfaces and allows queries from `10.42.42.0/24`.

## Where this sits in my lab

wg-easy runs on `proxmox2` alongside the rest of the networking stack. One UDP port forwarded at the router, and that is the entire inbound attack surface of this lab.

Clients point at AdGuard for DNS, which means my phone gets ad-blocking and internal hostname resolution anywhere. Everything else — Portainer, the *arr stack, Uptime Kuma, Grafana — stays firmly on the LAN with no external exposure at all, and is reached over this tunnel. [Seerr](seerr.md) is the one deliberate exception, because the people making requests are not going to install a VPN.
