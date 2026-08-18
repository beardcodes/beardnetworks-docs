# Proxmox VE

Everything else in this homelab sits on top of Proxmox. It is the hypervisor that runs the VMs and LXC containers, which in turn run Docker, which in turn runs about thirty services. Get this layer wrong and every layer above it inherits the problem, so it is worth spending an afternoon on.

This guide covers a clean Proxmox VE 9 install on a single node: writing the ISO, first boot, getting rid of the subscription nag, setting up storage, and creating the VM that will host your Docker stack.

[Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview) is a Debian-based, open-source virtualisation platform. It combines two hypervisor technologies in one web UI:

- **KVM** for full virtual machines — Windows, Ubuntu, TrueNAS, anything with its own kernel.
- **LXC** for system containers — lightweight Linux instances that share the host kernel and boot in about a second.

Other things that make it a good homelab foundation:

1. Web UI on port 8006, no client software to install
2. ZFS, LVM-thin, Ceph, NFS and SMB storage backends out of the box
3. Snapshots and scheduled backups built in, no add-on required
4. Clustering — add a second node later and manage both from one UI
5. Full API and a `pct` / `qm` CLI, so everything is scriptable
6. No licence cost; the paid subscription only buys the enterprise repo and support

## VM or LXC for Docker?

Both work. The split I use:

| | LXC container | Virtual machine |
|---|---|---|
| Boot time | ~1 second | ~20 seconds |
| RAM overhead | Almost none | 0.5–1 GB for the guest kernel |
| Docker support | Works, but needs `nesting=1` and `keyctl=1` | Officially supported, no tweaks |
| Snapshot/restore | Fast | Slower, but captures the whole disk |
| GPU passthrough | Easier (shared kernel) | Needs IOMMU and a bound device |

**Docker goes in a VM.** Docker-in-LXC works right up until a kernel update, a storage driver change, or an `overlayfs` quirk breaks it in a way that is genuinely hard to debug. The extra gigabyte of RAM buys you a supported configuration and a clean boundary. Use LXC for single-purpose Linux services where you want the density.

!!! note "Prerequisites"

    - A 64-bit machine with hardware virtualisation (Intel VT-x or AMD-V) enabled in the BIOS
    - 8 GB RAM minimum, 32 GB or more if you are running the stack in this repo
    - An SSD for the OS, plus whatever you intend to use for VM disks
    - An 8 GB or larger USB stick
    - A wired network connection — do not install a hypervisor over Wi-Fi

## 1. Download the ISO

Grab the installer ISO from the [Proxmox downloads page](https://www.proxmox.com/en/downloads/proxmox-virtual-environment). At the time of writing the current release is **Proxmox VE 9.2-1**, which is built on Debian 13 "Trixie".

Verify the checksum before you write it. The download page lists the SHA-256:

```bash
sha256sum proxmox-ve_9.2-1.iso
```

Compare that string to the one on the website. If it does not match, download it again.

## 2. Write the ISO to a USB stick

On Windows, use [Rufus](https://rufus.ie/) in **DD mode**. Rufus will offer ISO mode first — say no. The Proxmox ISO is a hybrid image and ISO mode produces a stick that boots into a broken installer.

On Linux or macOS, use `dd` directly:

```bash
sudo dd if=proxmox-ve_9.2-1.iso of=/dev/sdX bs=1M status=progress oflag=sync
```

!!! danger "Check the device name"

    `/dev/sdX` must be your USB stick, not your system disk. Run `lsblk` first and confirm the size matches the stick. `dd` will not ask twice, and it will not ask nicely.

## 3. Install

Boot from the stick and pick **Install Proxmox VE (Graphical)**. The installer asks four things:

**Target disk.** Pick your SSD. Click **Options** to choose the filesystem:

- `ext4` on LVM — the default, fine for a single disk
- `zfs (RAID1)` — pick this if you have two identical disks and want the OS mirrored

If you choose ZFS, set `ashift=12` for any modern drive and cap the ARC later (step 6), or ZFS will happily eat half your RAM.

**Location and timezone.** Sets the keyboard layout too, so get this right or the password screen will surprise you.

**Password and email.** The password is for the `root@pam` account you will log in with. The email is where backup and disk failure notifications go — use a real one.

**Network.** This is the important screen:

- **Hostname (FQDN)**: must be a fully qualified name, e.g. `pve.home.lan`. A bare `pve` is rejected.
- **IP address**: give it a static address in CIDR form, e.g. `192.168.1.10/24`. A hypervisor on DHCP is a hypervisor you cannot find after a power cut.
- **Gateway** and **DNS**: your router, or your AdGuard box if you have one running already.

Confirm the summary and let it install. It reboots into the shell and prints the URL to use.

## 4. First login and the subscription notice

Browse to `https://192.168.1.10:8006` — note **https** and port **8006**. You will get a certificate warning because the cert is self-signed; accept it for now.

Log in with user `root`, realm **Linux PAM standard authentication**.

You will immediately get a "No valid subscription" popup. Proxmox without a subscription is fully functional, but it points at the enterprise repository by default, which returns a 401 on every update. Fix both.

In **Datacenter → your node → Updates → Repositories**, disable the two `enterprise` entries and add the **No-Subscription** repository. Or from the shell:

```bash
# Disable the enterprise repos
sed -i 's/^Components: pve-enterprise/Components: pve-no-subscription/' /etc/apt/sources.list.d/pve-enterprise.sources
sed -i 's/^Enabled: true/Enabled: false/' /etc/apt/sources.list.d/ceph.sources

# Add the no-subscription repo
cat > /etc/apt/sources.list.d/pve-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt full-upgrade -y
reboot
```

!!! warning "The no-subscription repo is not the test repo"

    `pve-no-subscription` is the free stable repo and is what you want. Do not enable `pvetest` on a machine you care about — that one is for people who enjoy finding bugs.

## 5. Set up storage

Out of the box you get `local` (ISOs, templates, backups) and `local-lvm` (VM disks). That is enough to start.

Upload an Ubuntu Server ISO under **local → ISO Images → Upload**, or pull it straight from the node shell without touching your laptop:

```bash
cd /var/lib/vz/template/iso
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso
```

If you have extra disks, add them under **Datacenter → Storage**. A ZFS pool for VM disks is created from **node → Disks → ZFS → Create: ZFS**.

## 6. Two settings worth changing now

**Cap the ZFS ARC** — only if you installed on ZFS. By default ZFS will use up to half your RAM for cache, which looks alarming and starves your VMs:

```bash
# 8 GB cap, in bytes
echo "options zfs zfs_arc_max=8589934592" > /etc/modprobe.d/zfs.conf
update-initramfs -u -k all
reboot
```

**Turn on backups.** Under **Datacenter → Backup → Add**, create a job: all guests, daily, mode **Snapshot**, retention of 7 dailies. It takes two minutes and it is the single highest-value thing on this page.

## 7. Create the Docker VM

**Create VM** in the top right, then:

| Tab | Setting |
|---|---|
| General | Name `docker01`, tick **Start at boot** |
| OS | Your Ubuntu Server ISO |
| System | Machine `q35`, BIOS `OVMF (UEFI)`, tick **Qemu Agent** |
| Disks | 64 GB minimum, `SSD emulation` on, `Discard` on |
| CPU | 4 cores, Type **host** |
| Memory | 8192 MB, and untick **Ballooning** if the workload is memory-sensitive |
| Network | Bridge `vmbr0`, model `VirtIO (paravirtualized)` |

CPU type **host** matters — the default `kvm64` hides modern instruction sets and measurably slows things down. The only reason to avoid it is live migration between nodes with different CPUs, which does not apply to a single-node lab.

Install Ubuntu as normal, then install the guest agent so Proxmox can see the VM's IP address and shut it down cleanly:

```bash
sudo apt update && sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

From here, follow the [Docker guide](../host-setup/docker.md) inside that VM, and everything in the Docker Apps section will have somewhere to run.

## Troubleshooting

**The web UI will not load.** You are probably on `http` or the wrong port. It is `https://<ip>:8006`. If that still fails, check from the physical console that `pveproxy` is running: `systemctl status pveproxy`.

**`apt update` returns 401 Unauthorized.** The enterprise repo is still enabled. Go back to step 4.

**VM will not start, "KVM virtualisation not available".** Hardware virtualisation is off in the BIOS. Look for VT-x, AMD-V, or SVM Mode.

**No network after install.** Check `/etc/network/interfaces`. The installer binds `vmbr0` to the NIC it detected at install time; if you moved the cable to a different port, update the `bridge-ports` line and run `ifreload -a`.

**Time is wrong and certificates fail.** `timedatectl set-ntp true`.

## Where this sits in my lab

This node is `proxmox`, the first of two. It runs the VM that hosts Portainer, Karakeep, Home Assistant, Change Detection, LubeLogger and Dockhand. The second node, `proxmox2`, carries the networking and observability stack — Traefik, AdGuard, Uptime Kuma, Prometheus and Grafana — so that a reboot of one box never takes down both DNS and the dashboard telling me DNS is down.

Storage-heavy media services live on a separate Unraid box, because Proxmox is a good hypervisor and a mediocre NAS.
