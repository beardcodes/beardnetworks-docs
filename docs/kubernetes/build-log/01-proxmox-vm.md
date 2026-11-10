# K3s Build Log, Part 1: The Proxmox VM and First Node

*November 2026 · A real build log, not a copy-paste production blueprint*

I already run a lot of Docker containers across my homelab. Docker Compose still makes sense for plenty of them. But I wanted to learn what managing applications through Kubernetes and GitOps actually feels like, rather than treating Kubernetes as another way to run a container.

The goal was small: build one K3s node on Proxmox, manage it from my Windows desktop, and add services one at a time. I wasn't trying to replace every existing Docker host in a weekend.

This series documents what I actually built, including the bits that failed and how I fixed them. The live manifests are in [beardcodes/k3s-homelab](https://github.com/beardcodes/k3s-homelab).

## The starting point

Proxmox was already running in my homelab, alongside separate Docker environments. I created a dedicated Ubuntu Server virtual machine for K3s instead of installing Kubernetes on a Docker host or directly on the Proxmox hypervisor.

| Setting | My setup |
|---|---|
| Hypervisor | Proxmox VE |
| VM | `k3s-node1` (VM ID 1001) |
| Guest OS | Ubuntu Server 24.04 |
| CPU | 4 vCPU |
| Memory | 6 GB |
| Disk | 40 GB |
| LAN address | `192.168.100.3` |
| Kubernetes | K3s v1.37.2+k3s1 |
| Datastore | Embedded etcd, initialized as a single server |
| Networking decision | Disable bundled Traefik; configure Traefik and Gateway API later |

The 6 GB allocation is enough to learn with, but it became a real constraint after I added monitoring. A single-node lab isn't highly available, and embedded etcd doesn't change that when there's only one server.

## 1. Create the Ubuntu VM

In Proxmox, I created a normal Linux VM with four CPU cores, 6 GB RAM, and a 40 GB virtual disk. I connected it to the LAN bridge so it would be reachable directly from my desktop.

I installed Ubuntu Server 24.04, gave it the hostname `k3s-node1`, and arranged for its LAN address to remain `192.168.100.3`. A stable IP matters because it becomes the Kubernetes API endpoint and, in this particular setup, the address used to reach Traefik later.

After the first boot, I installed updates and basic tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

The guest agent gives Proxmox better visibility into the VM and makes clean shutdowns easier. I also verified SSH access before continuing.

## 2. Choose a deliberate K3s configuration

K3s bundles several conveniences, including Traefik by default. I wanted the routing layer to be represented in my GitOps repository, with a separate Gateway API configuration, so I disabled the bundled Traefik installation.

I also chose embedded etcd from the beginning rather than the default SQLite datastore. That doesn't give this single-server cluster fault tolerance; it just establishes the datastore layout for the lab.

The equivalent installation pattern is:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --cluster-init --disable traefik" sh -
```

!!! note "Command note"

    This reproduces the key configuration choices, not a verbatim copy of my original shell history. If you're following it yourself, check the current [K3s installation documentation](https://docs.k3s.io/installation) and configure the API certificate SAN (`--tls-san`) for the address your remote clients will use.

After installation, I verified the service and node:

```bash
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
```

At this stage, I wanted the node to be `Ready` and the core system pods healthy. I did not add applications yet.

## 3. Manage the cluster from Windows

Most of my day-to-day work happens on Windows in VS Code and PowerShell. I set up `kubectl` locally and kept a dedicated kubeconfig at:

```text
$HOME\.kube\k3s-node1.yaml
```

K3s writes its administrator kubeconfig to `/etc/rancher/k3s/k3s.yaml` on the server. I copied it to Windows through a secure transfer, changed the API server address from localhost to the VM's reachable address, and verified the connection.

```powershell
$env:KUBECONFIG = "$HOME\.kube\k3s-node1.yaml"
kubectl get nodes -o wide
kubectl get pods -A
```

!!! warning "Security note"

    That kubeconfig contains powerful cluster credentials. It does not belong in a public GitHub repository or a blog code example. I also had to ensure that the K3s API certificate was valid for the address used from Windows, rather than bypassing certificate checks.

## 4. What I decided not to do yet

I kept the initial architecture intentionally small. No attempt to migrate databases, no second worker, and no assumption that a persistent volume automatically counts as a backup.

I also kept my existing Docker services running. Kubernetes would be a parallel environment while I learned how to manage applications declaratively.

## 5. Where the build went next

Once `kubectl` worked from my desktop, I bootstrapped Flux CD against a dedicated GitHub repository: [beardcodes/k3s-homelab](https://github.com/beardcodes/k3s-homelab).

That introduced the part I actually wanted to learn: changing YAML in Git, pushing a commit, and having the cluster reconcile itself instead of manually applying every application.

The sequence after that was Podinfo for a first test deployment, Traefik with Kubernetes Gateway API, cert-manager with a Cloudflare DNS-01 wildcard certificate, then Uptime Kuma, Prometheus/Grafana, Homepage, and finally a read-only Proxmox API integration.

Along the way I hit Flux reconciliation timeouts, Grafana startup delays, an unwritable Homepage configuration mount, a bad Kubernetes volume name, and a Proxmox API token that kept returning HTTP 401. Those are worth documenting because they were more instructive than the successful YAML.

## What I learned from the first node

K3s itself was not the hard part. The decisions around it mattered more: where the VM lives, how it is addressed, how you access the API, whether routing is bundled or managed separately, and what belongs in Git.

A single-node K3s lab is a good place to learn those boundaries. It isn't a substitute for proper redundancy and backups, and 6 GB of RAM leaves less headroom than you'd think once monitoring is installed.

*Next: Part 2 — Bootstrapping Flux CD and deploying my first app from GitHub.*
