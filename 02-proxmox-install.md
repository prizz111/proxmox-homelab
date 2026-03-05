# 02 — Proxmox VE Installation

Install Proxmox VE 8 on all 3 nodes. You'll create one bootable USB and use it to install on each machine sequentially.

**Install on pve1 first, then pve2, then pve3.**

---

## Why This Matters

Proxmox VE (Virtual Environment) is a **Type 1 hypervisor** — unlike VirtualBox or VMware Workstation that run inside Windows, Proxmox runs directly on the bare metal. Your 3 machines will no longer run Windows or Linux as a desktop; Proxmox *is* the operating system. You manage everything through a web browser.

Proxmox is built on **Debian Linux**, which is why Linux commands and tools work directly on it.

---

## What You Need

- 1× USB drive (8GB minimum, 16GB+ recommended)
- A Windows PC with internet access to download and flash the ISO
- Monitor + keyboard (temporarily, for the install)

---

## Step 1: Download Proxmox VE 8

1. Go to [proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Under **Proxmox Virtual Environment**, click **ISO Images**
3. Download the latest **Proxmox VE 8.x ISO Installer** (approximately 1.1GB)

---

## Step 2: Create a Bootable USB

### Option A: Rufus (Windows — recommended)

1. Download [Rufus](https://rufus.ie) (free, portable, no install needed)
2. Insert your USB drive
3. Open Rufus:
   - **Device**: Select your USB drive
   - **Boot selection**: Click **SELECT** and choose the Proxmox ISO
   - **Partition scheme**: `GPT`
   - **Target system**: `UEFI (non CSM)`
4. Click **START** → when prompted about ISO mode, choose **Write in DD Image mode**
5. Wait for it to complete (~5 minutes)

> **Warning**: This erases everything on the USB drive. Back up anything on it first.

### Option B: Balena Etcher (Windows/Mac/Linux)

1. Download [Balena Etcher](https://etcher.balena.io)
2. Flash the Proxmox ISO to your USB drive
3. Etcher handles everything automatically (also uses DD mode)

---

## Step 3: Install Proxmox on Node 1 (pve1)

1. Insert the USB drive into the HP EliteDesk 800 G5
2. Power on and press **F9** to open the Boot Menu (or let it boot from USB per your BIOS settings)
3. Select your USB drive from the boot menu

The Proxmox installer will load. This takes about 30–60 seconds.

### In the Installer:

**Welcome screen** → Click **Install Proxmox VE (Graphical)**

**License Agreement** → Agree and Continue

**Target Disk** (critical step):
- Click the dropdown under **Target Harddisk**
- Select your **M.2 NVMe** drive (this is your OS drive — it will be wiped)
- Do **not** select your SATA HDD — that's for data later
- Click **Options** to review:
  - Filesystem: `ext4` (fine for homelab) or `zfs` (more advanced; skip for now)
  - Leave defaults and click **OK**

**Location and Time Zone**:
- Country: United States (or your country)
- Time zone: Your timezone
- Keyboard layout: U.S. English

**Administration Password and Email**:
- Set a strong root password (write it down — you'll use this to log into every node)
- Email: enter an email (used for system alerts; can be a placeholder)

**Network Configuration** (set a unique IP for each node):

For **pve1**:
| Field | Value |
|---|---|
| Management Interface | `eno1` (the Intel Ethernet port) |
| Hostname (FQDN) | `pve1.homelab.local` |
| IP Address | `192.168.1.101` |
| Netmask | `255.255.255.0` |
| Gateway | `192.168.1.1` (your router's IP) |
| DNS Server | `192.168.1.1` (your router, or use `1.1.1.1`) |

> **Note**: Adjust `192.168.1.1` if your router uses a different subnet (e.g., `10.0.0.1`).

**Summary screen** → Review everything, then click **Install**

Installation takes about 5–10 minutes. The machine will reboot automatically.

---

## Step 4: First Boot

After reboot, the HP EliteDesk will boot into Proxmox. You'll see a text console with:

```
Welcome to the Proxmox Virtual Environment.
Please use your web browser to configure this server -
connect to: https://192.168.1.101:8006/
```

You can now remove the USB drive.

> Proxmox doesn't have a graphical desktop — that's intentional. Everything is managed via the web UI.

---

## Step 5: Access the Web UI

From any computer on your home network:

1. Open a browser and go to `https://192.168.1.101:8006`
2. You'll get a **SSL certificate warning** — this is normal (Proxmox uses a self-signed cert by default). Click **Advanced** → **Proceed** (or equivalent in your browser)
3. Log in with:
   - Username: `root`
   - Password: (the one you set during install)
   - Realm: `Linux PAM standard authentication`
4. You'll see the Proxmox dashboard. You'll also see a **"No valid subscription"** warning — click OK to dismiss it (more on this below)

---

## Step 6: Post-Install Configuration (Each Node)

Run these commands on each node after install. You can do this via the web UI terminal or via SSH.

### Access the Shell

In the Proxmox web UI: click on your node (e.g., `pve1`) in the left sidebar → click **Shell**

### Disable the Enterprise Repository

Proxmox includes a paid enterprise update repository by default. For home use, switch to the free no-subscription repository.

```bash
# Disable the enterprise repo
echo "# enterprise repo disabled" > /etc/apt/sources.list.d/pve-enterprise.list

# Add the no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Also disable Ceph enterprise repo (not using Ceph, but avoids errors)
echo "# ceph enterprise repo disabled" > /etc/apt/sources.list.d/ceph.list
```

### Update All Packages

```bash
apt update && apt full-upgrade -y
```

This may take a few minutes. Reboot afterward:

```bash
reboot
```

---

## Step 7: Install on Node 2 and Node 3

Repeat Steps 3–6 for pve2 and pve3, **using different IPs and hostnames**:

**For pve2:**
- Hostname: `pve2.homelab.local`
- IP Address: `192.168.1.102`

**For pve3:**
- Hostname: `pve3.homelab.local`
- IP Address: `192.168.1.103`

Use the same root password on all 3 nodes (required for cluster joining later).

---

## Verification

After all 3 nodes are installed:

- [ ] `https://192.168.1.101:8006` loads Proxmox web UI for pve1
- [ ] `https://192.168.1.102:8006` loads Proxmox web UI for pve2
- [ ] `https://192.168.1.103:8006` loads Proxmox web UI for pve3
- [ ] All 3 nodes show up-to-date after `apt update && apt full-upgrade`
- [ ] No-subscription repo is active (no errors on `apt update`)

---

## Understanding What Just Happened

- **Proxmox on NVMe**: The NVMe drive now holds the Proxmox OS, including a `local-lvm` storage pool for VM disk images and a `local` storage area for ISOs and backups
- **SATA HDD untouched**: The SATA HDD is installed but unformatted — you'll configure it in [Guide 05](05-storage-setup.md)
- **No GUI desktop**: Proxmox is headless — the web UI at port 8006 is your control plane
- **Port 8006**: This is Proxmox's dedicated web interface port, served over HTTPS

---

## Next Step

Proceed to [03.1 - Switch Setup](03.1-switch-setup.md) to set up your UniFi Flex Mini before configuring the network.
