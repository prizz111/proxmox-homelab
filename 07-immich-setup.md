# 07 — Immich Setup

Deploy Immich, a self-hosted Google Photos alternative, as a virtual machine on your Proxmox cluster.

---

## Why This Matters

**Immich** backs up photos and videos from your phone automatically, gives you a Google Photos-style interface, and runs entirely on your hardware — your photos never leave your home network (or your Tailscale network). It supports face recognition, object detection, smart search, and albums.

We'll run Immich inside an **Ubuntu Server VM** on pve1, using Docker Compose to manage Immich's multiple containers (web server, machine learning, database, cache). The VM's media library will be stored on pve1's SATA HDD.

---

## Why a VM Instead of an LXC Container?

Proxmox supports two ways to run workloads:
- **VMs (Virtual Machines)**: Full virtualization — each VM has its own kernel. More overhead but maximum compatibility.
- **LXC Containers**: Lightweight containers sharing the host kernel. Less overhead but some software (especially Docker) is tricky inside LXC.

Immich uses Docker Compose internally, and running Docker inside LXC has known issues. A VM is the straightforward, well-supported path for Immich on Proxmox.

---

## Step 1: Download Ubuntu Server ISO

First, download the Ubuntu Server 24.04 LTS ISO onto pve1's local storage.

In the Proxmox web UI:
1. Click **pve1** → **local** (or **local-sata-pve1**) → **ISO Images**
2. Click **Download from URL**
3. Enter the Ubuntu Server 24.04 ISO URL:
   ```
   https://releases.ubuntu.com/24.04/ubuntu-24.04.2-live-server-amd64.iso
   ```
4. Click **Query URL** to verify, then **Download**

Wait for the download to complete (about 2.5GB).

> Alternatively, download the ISO to your PC and use **Upload** instead.

---

## Step 2: Create the Ubuntu VM

In the Proxmox web UI, click **Create VM** (top right button).

Work through each tab:

### General
| Field | Value |
|---|---|
| Node | `pve1` |
| VM ID | `100` (or any unused ID) |
| Name | `immich` |

### OS
| Field | Value |
|---|---|
| ISO Image | `ubuntu-24.04.2-live-server-amd64.iso` |
| Type | `Linux` |
| Version | `6.x - 2.6 Kernel` |

### System
| Field | Value |
|---|---|
| Machine | `q35` |
| BIOS | `OVMF (UEFI)` |
| Add EFI Disk | ✅ (on `local-lvm`) |
| TPM | Leave unchecked |

### Disks
| Field | Value |
|---|---|
| Bus/Device | `SCSI` |
| Storage | `local-sata-pve1` |
| Disk size | `64` (GB) — for Ubuntu OS + Docker images |

> Using the SATA HDD storage here keeps the NVMe free. The media files will be in a separate directory on the same disk.

### CPU
| Field | Value |
|---|---|
| Sockets | `1` |
| Cores | `4` |
| Type | `host` (exposes real CPU features — better performance) |

### Memory
| Field | Value |
|---|---|
| Memory | `8192` (8GB) |

Immich's machine learning features use significant RAM. 8GB is comfortable; 6GB is the minimum.

### Network
| Field | Value |
|---|---|
| Bridge | `vmbr0` |
| Model | `VirtIO (paravirtualized)` |

### Confirm
Review the summary and click **Finish**.

---

## Step 3: Install Ubuntu Server

1. Select the `immich` VM in the left panel
2. Click **Start** then **Console** to open the display
3. Ubuntu installer will load — use keyboard to navigate

**Key installation choices:**

- **Language**: English
- **Keyboard**: Your layout
- **Installation type**: Ubuntu Server (not minimized)
- **Network**: DHCP is fine for now (we'll set static IP after)
- **Disk**: Use entire disk, no LVM (since we want the full 64GB)
- **Username**: `homelab` (or your choice)
- **Password**: set a strong password
- **OpenSSH Server**: **Install** ✅ — important for remote access
- **Featured Snaps**: Skip (don't install any)

Click **Install Now** → wait ~10 minutes → **Reboot**

After reboot, the VM console will show the Ubuntu login prompt.

---

## Step 4: Post-Install Network and SSH Setup

Log in to the VM console with your credentials, then set a static IP:

```bash
# Find the network interface name
ip link show
# Usually named 'ens18' or similar in Proxmox VMs
```

Edit the netplan config:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace the contents with:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.1.110/24
      routes:
        - to: default
          via: 192.168.1.254
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

> Replace `ens18` with your actual interface name, and adjust IPs to match your network.

Apply the config:

```bash
sudo netplan apply
```

Verify connectivity:

```bash
ping 8.8.8.8 -c 3
```

You can now SSH into the Immich VM from your home network:

```bash
ssh homelab@192.168.1.110
```

---

## Step 5: Install Docker

From your SSH session into the Immich VM:

```bash
# Remove any old Docker packages
sudo apt remove docker docker-engine docker.io containerd runc 2>/dev/null

# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to the docker group (avoids needing sudo for docker commands)
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker works
docker run hello-world
```

---

## Step 6: Create the Media Directory

Immich stores uploaded photos and videos in a directory. We'll put this on the VM's disk (which is on the SATA HDD):

```bash
sudo mkdir -p /mnt/immich-media
sudo chown $USER:$USER /mnt/immich-media
```

---

## Step 7: Install Immich

```bash
# Create Immich directory
mkdir ~/immich && cd ~/immich

# Download the official docker-compose file and example env
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

Edit the `.env` file to configure where photos are stored:

```bash
nano .env
```

Key settings to update:

```bash
# Path where Immich stores uploaded media
UPLOAD_LOCATION=/mnt/immich-media

# Path for Immich's database (keep this on fast storage if possible)
DB_DATA_LOCATION=/home/homelab/immich/postgres

# Timezone (update to your timezone)
TZ=America/Chicago
```

Save the file (`Ctrl+X`, `Y`, `Enter`).

Start Immich:

```bash
docker compose up -d
```

This pulls all the required Docker images and starts the Immich containers. First run takes a few minutes.

Check that all containers started:

```bash
docker compose ps
```

You should see containers for: `immich_server`, `immich_machine_learning`, `immich_postgres`, `immich_redis` — all with status `Up`.

---

## Step 8: Access Immich

From your home network, open a browser and go to:

```
http://192.168.1.110:2283
```

You'll see the Immich first-run setup:
1. Create an admin account (email + password)
2. Log in
3. The dashboard is empty — upload some photos or set up the mobile app

### Mobile App Setup

1. Install the Immich app from the App Store or Google Play
2. Open the app → tap **Server URL**
3. Enter: `http://192.168.1.110:2283`
4. Log in with your admin credentials
5. Enable **Background Backup** to auto-sync your camera roll

---

## Step 9: Access Immich Remotely via Tailscale

To reach Immich from outside your home network, install Tailscale inside the Immich VM:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Authenticate via the URL shown. Once connected, the Immich VM gets a `100.x.x.x` Tailscale IP.

In the Immich mobile app, you can update the server URL to the Tailscale IP:
```
http://100.x.x.x:2283
```

Now your photos back up even when you're away from home.

---

## Step 10: Configure Auto-Start

Immich should start automatically when the VM boots:

```bash
# Make Docker start on boot (already default), and set Immich to always restart
cd ~/immich
```

The `docker compose up -d` command with Immich's default `restart: always` policy in `docker-compose.yml` already handles auto-restart. Verify the VM itself starts on boot:

In Proxmox web UI → click on the `immich` VM → **Options** → **Start at boot** → set to **Yes**

---

## Updating Immich

Immich updates frequently. To update:

```bash
cd ~/immich
docker compose pull
docker compose up -d
```

---

## Key Immich Features to Explore

| Feature | Where to Find |
|---|---|
| Face recognition | Explore → People |
| Smart search | Search bar (natural language: "sunset beach 2023") |
| Albums | Albums → Create |
| Sharing | Share albums with other Immich users |
| Map view | Explore → Places |
| Hardware transcoding | Admin → Video Transcoding |
| Backup status | App → Profile → Backup |

---

## Checklist

- [ ] Ubuntu Server 24.04 VM created on pve1 with 4 vCPU, 8GB RAM, 64GB disk on `local-sata-pve1`
- [ ] Ubuntu installed with OpenSSH server
- [ ] Static IP `192.168.1.110` configured
- [ ] Docker installed and working (`docker run hello-world` succeeds)
- [ ] `/mnt/immich-media` directory created
- [ ] Immich `docker-compose.yml` and `.env` downloaded
- [ ] `.env` configured with correct `UPLOAD_LOCATION` and timezone
- [ ] `docker compose up -d` completed, all containers running
- [ ] Immich accessible at `http://192.168.1.110:2283`
- [ ] Admin account created in Immich
- [ ] Tailscale installed in Immich VM for remote access
- [ ] VM set to start at boot in Proxmox

---

## What You've Built

Congratulations — your homelab is complete. Here's what you now have:

```
3-Node Proxmox Cluster (homelab-cluster)
├── pve1 (192.168.1.11 | 10.10.10.1 | 100.x.x.x Tailscale)
│   ├── Proxmox VE 8 (bare metal hypervisor)
│   ├── SATA HDD: local-sata-pve1 (bulk storage)
│   └── VM: immich (192.168.1.110)
│       ├── Ubuntu Server 24.04
│       ├── Docker + Immich
│       └── Tailscale (remote access)
├── pve2 (192.168.1.12 | 10.10.10.2 | 100.x.x.x Tailscale)
│   ├── Proxmox VE 8
│   └── SATA HDD: local-sata-pve2 (available for future VMs)
└── pve3 (192.168.1.13 | 10.10.10.3 | 100.x.x.x Tailscale)
    ├── Proxmox VE 8
    └── SATA HDD: local-sata-pve3 (available for future VMs)
```

**What you've learned:**
- Type 1 hypervisor setup and management
- 3-node cluster quorum and high availability concepts
- VLAN configuration on a managed switch
- Corosync cluster networking
- Linux storage management (fdisk, ext4, fstab)
- WireGuard-based VPN with Tailscale
- Docker Compose deployments
- Self-hosted application management

**Ideas for what to run next on pve2/pve3:**
- [Home Assistant](https://www.home-assistant.io) — smart home automation
- [Nextcloud](https://nextcloud.com) — self-hosted Google Drive
- [Jellyfin](https://jellyfin.org) — media server (movies, TV, music)
- [Pi-hole](https://pi-hole.net) — network-wide ad blocking
- [Vaultwarden](https://github.com/dani-garcia/vaultwarden) — self-hosted Bitwarden password manager
