# Windows and Debian VMs

Proxmox is installed and the node is up. Now you need guests on it. This page covers the two I build most: a **Windows 11** desktop VM for the things that refuse to run anywhere else, and a **Debian 12** VM that becomes a reusable cloud-init template.

They are different jobs. Windows is a one-off you hand-build, click through an installer, and keep for years. Debian is a thing you want ten of, so you build it once properly and clone it in four seconds after that.

The defaults Proxmox offers you are safe rather than fast. Both walkthroughs below change them.

!!! note "Prerequisites"

    - A working Proxmox VE node (see [Proxmox VE install](proxmox-ve-install.md))
    - Enough free space on `local-lvm` or your ZFS pool — 64 GB for Windows, 16 GB for Debian
    - For Windows 11: a CPU the host exposes as modern, plus a TPM device (Proxmox emulates one, you don't need hardware)

## Which Debian: 11 or 12?

**Use Debian 12 "bookworm".** It is the current stable release and gets full security support.

Debian 11 "bullseye" left standard support in August 2024. It is now on LTS, which is community-run, covers a reduced package set, and ends mid-2026. The only reason to build a bullseye VM today is an appliance or vendor agent that hard-refuses to run on bookworm — and if you hit that, everything below works identically, you just swap the ISO and the codename in the sources.

If you are reading this after Debian 13 "trixie" has shipped as stable, use that instead; nothing in this guide is bookworm-specific except the codename.

## 1. Get the ISOs onto the node

Don't upload from your laptop over Wi-Fi. Pull them on the node itself, straight into the ISO store:

```bash
cd /var/lib/vz/template/iso

# Debian 12 netinst — check the download page for the current point release
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.11.0-amd64-netinst.iso

# VirtIO drivers for Windows — always grab the stable-latest
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

The Debian point release number changes every few months and the old file disappears from `current/`. If `wget` 404s, open the [Debian netinst page](https://www.debian.org/distrib/netinst) and take the filename from there.

Windows ISOs you have to fetch from Microsoft yourself — the [Windows 11 download page](https://www.microsoft.com/software-download/windows11) gives you a direct ISO link. Upload it via **local → ISO Images → Upload**, or `wget` the signed URL on the node before it expires.

Verify the Debian image while you are there. Debian publishes signed checksums next to the ISO:

```bash
wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

---

## Part one: the Windows 11 VM

Windows on KVM is fine once it has paravirtualised drivers. Without them it will not see a VirtIO disk during setup, and you get a screen saying no drives were found — which is the single most common reason people give up here.

### 2. Create the VM

**Create VM**, top right, and change these away from the defaults:

| Tab | Setting | Why |
|---|---|---|
| General | Name `win11`, untick **Start at boot** unless you really want it | A desktop VM that boots with the node eats RAM you probably wanted elsewhere |
| OS | Your Windows 11 ISO, Guest OS type **Microsoft Windows 11/2022/2025** | Sets sane hidden defaults |
| System | Graphic card **Default**, Machine **q35**, BIOS **OVMF (UEFI)**, tick **Add EFI Disk**, tick **Add TPM**, TPM version **v2.0** | Windows 11 refuses to install without UEFI, Secure Boot and a TPM |
| System | SCSI Controller **VirtIO SCSI single** | |
| Disks | Bus **SCSI**, 64 GB minimum, Cache **Write back**, tick **Discard** and **SSD emulation** | Discard lets deleted space return to the pool |
| CPU | 4 cores, Type **host** | `kvm64` hides half the instruction set and Windows feels it |
| Memory | 8192 MB, untick **Ballooning** | Windows and the balloon driver argue; give it a fixed number |
| Network | Bridge `vmbr0`, Model **VirtIO (paravirtualized)** | |

Do **not** start it yet.

### 3. Attach the driver ISO

The installer needs the VirtIO storage driver before it can see the disk you just made, so give it a second optical drive.

**win11 → Hardware → Add → CD/DVD Drive**, bus `IDE 3`, and select `virtio-win.iso`.

### 4. Install

Start the VM and open the console fast — the UEFI "Press any key to boot from CD" prompt times out in about five seconds, and if you miss it you land in the EFI shell. Type `exit`, choose **Boot Manager**, pick the DVD drive, and try again.

Click through until **"Where do you want to install Windows?"** shows an empty list. That is expected:

1. **Load driver → Browse**
2. Go to the virtio CD → `vioscsi` → `w11` → `amd64`
3. Next. The disk appears.

While you are in there, load two more from the same CD — it saves a reboot later:

- `NetKVM\w11\amd64` — the network card
- `Balloon\w11\amd64` — memory reporting

Install as normal.

!!! tip "Skipping the Microsoft account"

    Windows 11 Home and recent Pro builds push hard for an online account. At the network or sign-in screen press ++shift+f10++ for a command prompt and run:

    ```text
    start ms-cxh:localonly
    ```

    That opens the local-account dialog directly. The older `oobe\bypassnro` trick was removed in 2025 builds — on those it either does nothing or reboots you into the same screen.

### 5. Guest tools

Once you are at the desktop, open the virtio CD in Explorer and run **`virtio-win-guest-tools.exe`**. That installs the remaining drivers plus the QEMU guest agent as a service.

Back in Proxmox, **Options → QEMU Guest Agent → Enabled**, then reboot the VM. The Summary tab should now show the guest's IP address, and **Shutdown** becomes a graceful shutdown instead of a five-second hold of the power button.

Two more things worth doing now:

- **Options → Use tablet for pointer: Yes** (default) if you use the web console; set it to **No** only if you connect exclusively over RDP, where it saves a little CPU.
- Enable RDP inside Windows and connect that way for daily use. The noVNC console is for emergencies — see [Remote Desktop](../host-setup/remote-desktop.md).

---

## Part two: the Debian 12 template

Build this once. Clone it forever.

### 6. Create the VM

Same **Create VM** dialog, different answers:

| Tab | Setting |
|---|---|
| General | Name `debian12-tpl` |
| OS | Debian netinst ISO, Guest OS type **Linux 6.x - 2.6 Kernel** |
| System | Machine **q35**, BIOS **OVMF (UEFI)**, **Add EFI Disk**, tick **Qemu Agent**, SCSI Controller **VirtIO SCSI single** |
| Disks | Bus **SCSI**, 16 GB, Cache **Write back**, **Discard** on, **SSD emulation** on |
| CPU | 2 cores, Type **host** |
| Memory | 2048 MB, Ballooning on with a minimum of 1024 MB |
| Network | Bridge `vmbr0`, Model **VirtIO (paravirtualized)** |

16 GB sounds small. It is a template — clones get resized up in one click, and a small template clones fast.

### 7. Install Debian

The netinst installer is short. The parts that matter:

- **Hostname** — anything, the clone will rename itself later
- **Partitioning** — *Guided – use entire disk*, all files in one partition. Do not create a separate `/home` on a template
- **Software selection** — this is the important screen. Untick **everything**, including the desktop environment, then tick only:
    - **SSH server**
    - **standard system utilities**

You want a VM that boots in eight seconds with no display stack in it.

### 8. Prepare it for cloning

Log in and get it into a state worth duplicating:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent cloud-init curl
sudo systemctl enable --now qemu-guest-agent
```

Then strip the identity out of it. A cloned machine that keeps the original's SSH host keys and machine ID will hand you duplicate DHCP leases and SSH warnings for weeks:

```bash
# Remove SSH host keys — sshd regenerates them on first boot
sudo rm -f /etc/ssh/ssh_host_*

# Blank the machine ID (the file must exist but be empty)
sudo truncate -s 0 /etc/machine-id
sudo rm -f /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

# Clear logs, apt cache, and shell history
sudo apt clean
sudo cloud-init clean --logs
sudo rm -rf /var/log/*.gz /var/log/*.1
cat /dev/null > ~/.bash_history && history -c

sudo shutdown -h now
```

!!! warning "Run these last, then power off"

    After deleting the host keys, do not boot this VM again as itself. If you do, it regenerates keys and you have baked new fixed identity back into the template. Next boot should always be a clone.

### 9. Convert to a template

Two changes in the Proxmox UI while it is off:

1. **Hardware → Add → CloudInit Drive**, storage `local-lvm`. This is how each clone receives its hostname, user, SSH key and IP.
2. **Hardware** → remove the CD/DVD drive, or set it to *Do not use any media*.

Then **right-click the VM → Convert to template**. The icon changes and the config becomes read-only.

### 10. Clone it

Right-click the template → **Clone**. Mode **Full Clone** for anything you care about (linked clones stay chained to the template forever). Set the new name and VM ID.

Before first boot, fill in the **Cloud-Init** tab on the clone:

| Field | Value |
|---|---|
| User | your username |
| Password | leave blank if you are using a key |
| SSH public key | paste your key — see [SSH](../host-setup/ssh.md) |
| IP Config (net0) | `192.168.1.50/24`, gateway `192.168.1.1`, or `DHCP` |

Resize the disk if you need more than 16 GB — **Hardware → Hard Disk → Disk Action → Resize**. Debian's cloud-init grows the filesystem to fill it on boot; you do not need to touch `parted`.

Start it. It comes up with your key installed, on the address you set, in well under a minute.

Or do the whole thing from the shell:

```bash
qm clone 9000 121 --name k3s-node1 --full
qm set 121 --ipconfig0 ip=192.168.1.51/24,gw=192.168.1.1
qm set 121 --sshkeys ~/.ssh/authorized_keys
qm resize 121 scsi0 +16G
qm start 121
```

## Troubleshooting

**Windows setup says no drives were found.** The VirtIO SCSI driver is not loaded. Step 4 — and make sure you picked the `w11` folder, not `w10`.

**Windows 11 install refuses: "This PC can't run Windows 11".** Missing TPM or the machine is on SeaBIOS. Shut down, then check **Hardware** for a TPM State device and **Options → BIOS → OVMF (UEFI)**. Adding a TPM after the EFI disk exists is fine.

**VM boots to `UEFI Interactive Shell`.** It missed the boot device. Type `exit`, choose **Boot Manager**, select the drive. If it happens every boot, the EFI boot entry is missing — in the EFI shell, `fs0:` then `cd EFI\BOOT` and run `BOOTX64.EFI` once, and the entry gets written.

**Proxmox shows no IP for the guest.** The guest agent is not running. Linux: `systemctl status qemu-guest-agent`. Windows: check the *QEMU Guest Agent* service. Also confirm **Options → QEMU Guest Agent** is enabled on the Proxmox side — both halves are needed.

**Every clone gets the same DHCP address.** They share a machine ID. Go back to step 8, redo it on the template, and re-clone.

**Clone boots with the template's hostname.** The cloud-init drive is missing from that clone. Add it under Hardware and reboot.

**Debian clone has no network.** Cloud-init writes `net0` config only if the CloudInit drive exists *and* the IP Config field is filled in. An empty IP Config means no config at all — set it explicitly to `DHCP` if that is what you want.

## What's next

The Debian template is the machine every other guide here starts from. Point it at [Docker](../host-setup/docker.md) for the container stack, or clone three of them and build a [K3s cluster](../kubernetes/k3s-installation.md).
