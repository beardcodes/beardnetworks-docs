---
hide:
  - navigation
---

<div class="bn-mast" markdown>

<span class="bn-byline">beardnetworks — homelab notes</span>

# I write down what worked, before I forget it

Everything here runs at home on hardware I paid for and broke myself. These are working notes: the compose file that actually came up, the flag that fixed the permission error, the step the official docs left out. Copy what's useful, ignore what isn't.

</div>

Read it top to bottom and you get the build order hypervisor first, then the host, then the containers, then the cluster.

<div class="bn-layer" markdown>
<div class="bn-layer__rail" markdown>layer 01 · metal</div>
<div markdown>

## Proxmox

The hypervisor everything else sits on. Install, storage layout, and the VM that ends up running the Docker stack.

- [Proxmox VE install](proxmox/proxmox-ve-install.md) — *bare metal to first VM*
- [Windows & Debian VMs](proxmox/windows-debian-vms.md) — *a hand-built Windows box, and a Debian template you clone forever*

</div>
</div>

<div class="bn-layer" markdown>
<div class="bn-layer__rail" markdown>layer 02 · host</div>
<div markdown>

## Host setup

What I do to every fresh box before anything gets deployed to it.

- [SSH](host-setup/ssh.md) — *keys, hardening, and not locking yourself out*
- [Docker](host-setup/docker.md) — *engine and compose on a clean host*
- [Mapped network drive](host-setup/mapped-network-drive.md) — *shares that survive a reboot*
- [Remote desktop](host-setup/remote-desktop.md) — *when SSH isn't enough*

</div>
</div>

<div class="bn-layer" markdown>
<div class="bn-layer__rail" markdown>layer 03 · containers</div>
<div markdown>

## Docker containers

The bulk of the lab. Each writeup has the compose file, the reverse proxy config, and the part that bit me.

- [Media stack](docker-containers/media-stack.md) — *Prowlarr, the \*arrs, downloaders, and Seerr as one unit*
- [Traefik](docker-containers/traefik.md) and [Nginx Proxy Manager](docker-containers/nginx-proxy-manager.md) — *getting traffic to the right container*
- [AdGuard](docker-containers/adguard.md), [wg-easy](docker-containers/wg-easy.md), [CrowdSec](docker-containers/crowdsec.md) — *DNS, VPN, and keeping the noise out*
- [Tinyauth](docker-containers/tinyauth.md) and [Vaultwarden](docker-containers/vaultwarden.md) — *one login, and somewhere to keep the rest*
- [Portainer](docker-containers/portainer.md), [Dockhand](docker-containers/dockhand.md), [Nexterm](docker-containers/nexterm.md) — *managing it all from a browser*
- [Uptime Kuma](docker-containers/uptime-kuma.md), [Prometheus](docker-containers/prometheus.md), [Grafana](docker-containers/grafana.md), [Diun](docker-containers/diun.md) — *knowing before the family does*
- [Glance](docker-containers/glance.md) and [Homepage](docker-containers/homepage.md) — *the tab that stays open*
- [Mealie](docker-containers/mealie.md), [Karakeep](docker-containers/karakeep.md), [Firefly III](docker-containers/firefly-iii.md), [IT-Tools](docker-containers/it-tools.md) — *the small useful ones*

</div>
</div>

<div class="bn-layer" markdown>
<div class="bn-layer__rail" markdown>layer 04 · cluster</div>
<div markdown>

## Kubernetes

Where the single-host stack goes once one host stops being enough.

- [Set up K3s](kubernetes/k3s-installation.md) — *a cluster small enough to understand*
- [Helm](kubernetes/helm-install.md) — *installing charts without hand-writing manifests*
- [cert-manager](kubernetes/cert-manager.md) — *wildcard TLS that renews itself*

</div>
</div>

<div class="bn-colophon" markdown>

Something wrong, out of date, or missing? Open an issue or a PR on [GitHub](https://github.com/beardcodes). If a guide saved you an evening, you can [buy me a coffee](https://www.buymeacoffee.com/beardnetwork).

</div>
