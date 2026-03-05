# 07 — Immich Setup

Deploy Immich, a self-hosted Google Photos alternative, as a virtual machine on your Proxmox cluster.

---

## Why This Matters

**Immich** backs up photos and videos from your phone automatically, gives you a Google Photos-style interface, and runs entirely on your hardware — your photos never leave your home network (or your Tailscale network). It supports face recognition, object detection, smart search, and albums.

We'll run Immich inside an **Ubuntu Server VM** on pve1, using Docker Compose to manage Immich's multiple containers. The VM's media library will be stored on pve1's SATA HDD.

---

## Why a VM Instead of an LXC Container?

Proxmox supports two ways to run workloads:
- **VMs**: Full virtualization — each VM has its own kernel. More overhead but maximum compatibility.
- **LXC Containers**: Lightweight, shares host kernel. Docker is tricky inside LXC.

Immich uses Docker Compose, and running Docker inside LXC has known issues. A VM is the straightforward path.

---

## Step 1: Download Ubuntu Server ISO

In the Proxmox web UI:
1. Click **pve1** → **local** → **ISO Images**
2. Click **Download from URL**
3. URL: `https://releases.ubuntu.com/24.04/ubuntu-24.04.2-live-server-amd64.iso`
4. Click **Query URL** → **Download**

---

## Step 2: Create the Ubuntu VM

Click **Create VM** in the Proxmox web UI:

| Section | Field | Value |
|---|---|---|
| General | Name | `immich` |
| OS | ISO Image | `ubuntu-24.04.2-live-server-amd64.iso` |
| System | Machine | `q35`, BIOS: `OVMF (UEFI)` |
| Disks | Storage | `local-sata-pve1`, Size: `64GB` |
| CPU | Cores | `4`, Type: `host` |
| Memory | Memory | `8192` (8GB) |
| Network | Bridge | `vmbr0`, Model: `VirtIO` |

Click **Finish**.

---

## Step 3: Install Ubuntu Server

1. Select the VM → **Start** → **Console**
2. Work through the Ubuntu installer:
   - Installation type: Ubuntu Server (not minimized)
   - Network: DHCP (set static IP after)
   - Username: `homelab`
   - **OpenSSH Server: Install ✅**
   - Featured Snaps: Skip
3. Wait ~10 minutes → Reboot

---

## Step 4: Set a Static IP

Log in via console, then:

```bash
ip link show   # note your interface name (usually ens18)
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace contents:

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
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

```bash
sudo netplan apply
ping 8.8.8.8 -c 3   # verify connectivity
```

You can now SSH from your home network: `ssh homelab@192.168.1.110`

---

## Step 5: Install Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world   # verify it works
```

---

## Step 6: Create the Media Directory

```bash
sudo mkdir -p /mnt/immich-media
sudo chown $USER:$USER /mnt/immich-media
```

---

## Step 7: Install Immich

```bash
mkdir ~/immich && cd ~/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

Edit `.env`:

```bash
nano .env
```

Key settings:

```bash
UPLOAD_LOCATION=/mnt/immich-media
DB_DATA_LOCATION=/home/homelab/immich/postgres
TZ=America/Chicago   # update to your timezone
```

Start Immich:

```bash
docker compose up -d
docker compose ps   # verify all containers are Up
```

---

## Step 8: Access Immich

Open `http://192.168.1.110:2283` in your browser.

1. Create an admin account
2. Install the Immich mobile app → set server URL to `http://192.168.1.110:2283`
3. Enable **Background Backup** to auto-sync your camera roll

---

## Step 9: Access Immich Remotely via Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

In the Immich app, update server URL to `http://100.x.x.x:2283` (the VM's Tailscale IP).

---

## Step 10: Configure Auto-Start

In Proxmox web UI → `immich` VM → **Options** → **Start at boot** → **Yes**

Docker's `restart: always` policy (set by default in Immich's compose file) handles container restart automatically.

---

## Updating Immich

```bash
cd ~/immich
docker compose pull
docker compose up -d
```

---

## Checklist

- [ ] Ubuntu Server 24.04 VM created on pve1 (4 vCPU, 8GB RAM, 64GB on `local-sata-pve1`)
- [ ] Ubuntu installed with OpenSSH server
- [ ] Static IP `192.168.1.110` configured
- [ ] Docker installed and working
- [ ] `/mnt/immich-media` directory created
- [ ] `.env` configured with correct `UPLOAD_LOCATION` and timezone
- [ ] `docker compose up -d` completed, all containers running
- [ ] Immich accessible at `http://192.168.1.110:2283`
- [ ] Admin account created
- [ ] Tailscale installed in Immich VM
- [ ] VM set to start at boot in Proxmox

---

## What You've Built

Your homelab is complete:

```
3-Node Proxmox Cluster (homelab-cluster)
├── pve1 (192.168.1.101 | 10.10.10.1 | Tailscale)
│   ├── Proxmox VE 8
│   ├── SATA HDD: local-sata-pve1
│   └── VM: immich (192.168.1.110)
│       ├── Ubuntu Server 24.04
│       ├── Docker + Immich
│       └── Tailscale
├── pve2 (192.168.1.102 | 10.10.10.2 | Tailscale)
│   └── SATA HDD: local-sata-pve2 (available for future VMs)
└── pve3 (192.168.1.103 | 10.10.10.3 | Tailscale)
    └── SATA HDD: local-sata-pve3 (available for future VMs)
```

**Ideas for what to run next on pve2/pve3:**
- [Home Assistant](https://www.home-assistant.io) — smart home automation
- [Nextcloud](https://nextcloud.com) — self-hosted Google Drive
- [Jellyfin](https://jellyfin.org) — media server (movies, TV, music)
- [Pi-hole](https://pi-hole.net) — network-wide ad blocking
- [Vaultwarden](https://github.com/dani-garcia/vaultwarden) — self-hosted Bitwarden password manager
