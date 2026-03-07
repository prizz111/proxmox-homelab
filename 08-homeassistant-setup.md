# 08 — Home Assistant Setup

Deploy Home Assistant OS as a VM on pve2, then migrate your smart home devices away from Google Assistant.

---

## Why This Matters

**Home Assistant** is a self-hosted smart home platform that runs entirely on your hardware. Unlike Google Assistant, it:
- Keeps all your smart home data local — nothing goes to Google's servers
- Works without internet (automations still run if your ISP goes down)
- Integrates with virtually every smart home brand in one unified interface
- Gives you full control over automations, dashboards, and device grouping

Your current devices (Hue, Kasa, Eufy, Google Minis, Chromecasts) all have Home Assistant integrations, and most of them (Hue, Kasa, Chromecasts) work **locally** — no cloud dependency at all.

---

## Why Home Assistant OS (HAOS)?

Home Assistant comes in several install flavors. We'll use **Home Assistant OS** — the official full image — because:
- It's the recommended and fully supported installation method
- Includes the supervisor, add-on store, and automatic updates out of the box
- Simplest path on Proxmox: import a pre-built disk image and boot

---

## Why pve2?

pve1 is running the Immich VM. Putting Home Assistant on pve2 distributes load across the cluster and keeps the two services independent.

---

## Network Address

| Host | IP |
|---|---|
| homeassistant VM | `192.168.1.111` |

---

## Step 1: Download the Home Assistant OS Image

On **pve2**, open a shell (Proxmox web UI → pve2 → Shell) and download the HAOS KVM image:

```bash
cd /tmp

# Download the latest HAOS image for KVM/QEMU (check https://github.com/home-assistant/operating-system/releases for latest version)
wget -O haos.qcow2.xz "https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-$(curl -s https://api.github.com/repos/home-assistant/operating-system/releases/latest | grep -oP '"tag_name": "\K[^"]+').qcow2.xz"

# Extract it
xz -d haos.qcow2.xz
ls -lh haos.qcow2  # should be ~1-2GB
```

> Alternatively, download the `.qcow2.xz` file to your PC from the [HAOS releases page](https://github.com/home-assistant/operating-system/releases), then upload it to pve2 via **pve2 → local → ISO Images → Upload** (it doesn't have to be an ISO).

---

## Step 2: Create the Home Assistant VM

In the Proxmox web UI, click **Create VM**.

### General
| Field | Value |
|---|---|
| Node | `pve2` |
| VM ID | `200` (or any unused ID) |
| Name | `homeassistant` |

### OS
| Field | Value |
|---|---|
| Do not use any media | ✅ (select "Do not use any media") |
| Type | `Linux` |
| Version | `6.x - 2.6 Kernel` |

### System
| Field | Value |
|---|---|
| Machine | `q35` |
| BIOS | `OVMF (UEFI)` |
| Add EFI Disk | ✅ (on `local-lvm`) — **uncheck "Pre-enroll keys"** |
| TPM | Leave unchecked |

> **Important:** "Pre-enroll keys" enables Secure Boot, which blocks HAOS from booting with an `Access Denied` error. Always leave it unchecked.

### Disks
Delete any disk that was auto-added — we'll import the HAOS image as the disk instead.

### CPU
| Field | Value |
|---|---|
| Sockets | `1` |
| Cores | `2` |
| Type | `host` |

### Memory
| Field | Value |
|---|---|
| Memory | `4096` (4GB) |

Home Assistant is lightweight. 4GB is comfortable; 2GB is the minimum.

### Network
| Field | Value |
|---|---|
| Bridge | `vmbr0` |
| Model | `VirtIO (paravirtualized)` |

### Confirm
Review and click **Finish** — but **do not start the VM yet**.

---

## Step 3: Import the HAOS Disk

Back in the pve2 shell, import the downloaded image as the VM's disk (replace `200` with your VM ID if different):

```bash
qm importdisk 200 /tmp/haos.qcow2 local-lvm
```

This takes a minute. When complete, go to the Proxmox web UI:

1. Click on the `homeassistant` VM → **Hardware**
2. You'll see an **Unused Disk 0** — double-click it
3. Set **Bus/Device** to `SCSI`, slot `0`
4. Click **Add**

Then set the boot order:
1. **Options** → **Boot Order**
2. Enable `scsi0` and move it to the top
3. Disable any other boot devices
4. Click **OK**

---

## Step 4: Start Home Assistant

Click **Start** on the VM. Open the **Console** to watch it boot.

Home Assistant OS will:
1. Boot into a minimal Linux environment
2. Download and install the latest Home Assistant version (requires internet — takes 5-10 minutes)
3. Start the Home Assistant web interface

When you see `homeassistant login:` in the console, it's ready. Do **not** log in here — Home Assistant is managed entirely via the web UI.

---

## Step 5: Access Home Assistant and Set Static IP

From your home network, open a browser and go to:

```
http://homeassistant.local:8123
```

If that doesn't resolve, check the VM console for the IP it got via DHCP, then navigate to `http://<that-ip>:8123`.

### Set a static IP

In Home Assistant web UI:
1. **Settings** → **System** → **Network**
2. Under your network interface, switch from **Automatic (DHCP)** to **Static**
3. Set:
   - IP Address: `192.168.1.111/24`
   - Gateway: `192.168.1.254` (your router)
   - DNS: `1.1.1.1, 8.8.8.8`
4. Click **Save**

Home Assistant will reconnect at the static IP. Navigate to `http://192.168.1.111:8123` going forward.

---

## Step 6: Initial Setup

The first-run wizard will walk you through:
1. **Create account** — name, username, password
2. **Home location** — used for sunrise/sunset automations
3. **Analytics** — opt in or out (your choice)
4. **Device discovery** — HA will already detect devices on your network

You'll likely see Philips Hue, Kasa devices, and Chromecasts listed immediately. You can add them here or skip and do it manually.

---

## Step 7: Add Integrations

Go to **Settings** → **Devices & Services** → **Add Integration** for each of the following.

### Philips Hue

Home Assistant discovers your Hue Bridge automatically.

1. Search for **Philips Hue** → select it
2. HA will find your bridge on the network
3. **Go press the button on your Hue Bridge**, then click **Submit** in HA
4. All your Hue bulbs and groups will appear as entities

> This is fully local — HA talks directly to your Hue Bridge, no Philips cloud involved.

### TP-Link Kasa

1. Search for **TP-Link Kasa** → select it
2. HA discovers Kasa devices automatically on your local network
3. If auto-discovery misses any, enter your TP-Link account credentials to pull the full device list
4. All Kasa plugs, bulbs, and switches appear as entities

> Local control — HA communicates directly with Kasa devices over your LAN.

### Google Cast (Chromecasts + Google Minis)

1. Search for **Google Cast** → select it
2. HA automatically discovers all Cast-capable devices (Chromecast TVs and Google Mini speakers)
3. They appear as **media player** entities

This lets you:
- See what's playing on each TV
- Control playback (pause, volume, etc.) from HA
- Send text-to-speech announcements through Google Mini speakers
- Trigger automations based on TV state (e.g., "when TV turns on, dim the lights")

> Note: Google Minis continue to work as Google Assistant speakers. HA uses them as audio output devices — they don't lose their Google Assistant functionality.

### Eufy Cameras

Eufy doesn't have an official HA integration. The community **eufy_security** integration works well but requires **HACS** (Home Assistant Community Store) first.

#### Install HACS

1. In HA, go to **Settings** → **Add-ons** → **Add-on Store**
2. Install the **Terminal & SSH** add-on and open a terminal
3. Run:
   ```bash
   wget -O - https://get.hacs.xyz | bash -
   ```
4. Restart Home Assistant: **Settings** → **System** → **Restart**
5. Go to **Settings** → **Devices & Services** → **Add Integration** → search **HACS**
6. Follow the GitHub authorization flow

#### Install eufy_security via HACS

1. In HACS → **Integrations** → search **eufy_security** → **Download**
2. Restart Home Assistant
3. **Settings** → **Devices & Services** → **Add Integration** → **Eufy Security**
4. Enter your Eufy account credentials
5. Your cameras appear as entities with live stream support

> Eufy cameras go through Eufy's cloud for authentication. Live streams can be routed locally once authenticated.

---

## Step 8: Migrate Devices Away from Google Assistant

You don't need to delete your Google Home setup immediately — you can run both in parallel while you get comfortable with HA. When you're ready to fully migrate:

### Smart Lights (Hue + Kasa)
- Control them entirely from the HA app and dashboard
- Remove them from Google Home: **Google Home app** → tap each device → **Settings** → **Unlink**
- Or leave them in Google Home — they'll still work, you'll just have two control surfaces

### Chromecasts + Google Minis
- Leave these in Google Home — they still function as Cast targets and Google Assistant speakers
- HA controls them in parallel; there's no conflict

### Eufy Camera
- View live feeds in the HA dashboard via the Camera card
- Set up motion-triggered automations in HA instead of Eufy's app
- The Eufy app still works alongside HA

---

## Step 9: Install the Mobile App

The Home Assistant companion app gives you:
- A mobile dashboard matching your web UI
- Location tracking for presence-based automations ("turn on lights when I arrive home")
- Push notifications from HA automations
- Quick access via Tailscale when away from home

1. Install **Home Assistant** from the App Store or Google Play
2. Open the app → **Get Started**
3. Enter server URL: `http://192.168.1.111:8123`
4. Log in with your HA credentials
5. Grant location permissions for presence detection

---

## Step 10: Remote Access via Tailscale

Install Tailscale inside the Home Assistant OS:

1. In HA → **Settings** → **Add-ons** → **Add-on Store**
2. Search **Tailscale** → **Install**
3. Start the add-on → open the **Web UI**
4. Authenticate via the Tailscale link shown

The HA VM gets a `100.x.x.x` Tailscale IP. Update the mobile app server URL to the Tailscale IP for remote access.

---

## Step 11: Configure Auto-Start

In Proxmox web UI → click on the `homeassistant` VM → **Options** → **Start at boot** → set to **Yes**.

---

## Step 12: Create Your First Automation

A simple example to verify everything is working — dim lights when a Chromecast starts playing:

1. **Settings** → **Automations & Scenes** → **Create Automation**
2. **Trigger**: State → entity: your Chromecast → to: `playing`
3. **Action**: Call service → `light.turn_on` → target your Hue/Kasa lights → brightness: 40%
4. Save and name it "Movie mode"

---

## Checklist

- [ ] HAOS `.qcow2` image downloaded to pve2
- [ ] VM created with 2 vCPU, 4GB RAM on pve2
- [ ] HAOS disk imported and set as boot device
- [ ] Home Assistant accessible at `http://192.168.1.111:8123`
- [ ] Static IP `192.168.1.111` configured
- [ ] Admin account created
- [ ] Philips Hue integration added (bridge button pressed)
- [ ] TP-Link Kasa integration added, all devices visible
- [ ] Google Cast integration added (Chromecasts + Google Minis visible)
- [ ] HACS installed
- [ ] eufy_security integration added, cameras visible
- [ ] Mobile app installed and connected
- [ ] Tailscale add-on installed for remote access
- [ ] VM set to start at boot in Proxmox

---

## Updating Home Assistant

Home Assistant updates frequently. When an update is available, a notification appears in the UI:

**Settings** → **System** → **Updates** → **Install**

Updates install in about 5 minutes and HA restarts automatically. Your configuration is preserved.

---

## What You've Built

```
3-Node Proxmox Cluster (homelab-cluster)
├── pve1 (192.168.1.11)
│   └── VM: immich (192.168.1.110)
├── pve2 (192.168.1.12)
│   └── VM: homeassistant (192.168.1.111)
│       ├── Philips Hue (local)
│       ├── TP-Link Kasa (local)
│       ├── Google Cast — Chromecasts + Minis
│       └── Eufy cameras
└── pve3 (192.168.1.13)
    └── (available for future VMs)
```

---

## Next Steps

Ideas for expanding your Home Assistant setup:
- **Dashboards**: Build custom Lovelace dashboards per room
- **Presence detection**: Use phone GPS or network device tracking for "home/away" automations
- **Energy monitoring**: Add TP-Link Kasa smart plugs with power monitoring to track appliance usage
- **Voice control**: Use a local voice assistant (Wyoming + Whisper add-on) to replace Google Assistant entirely
- **Notifications**: Push alerts to your phone for motion detection, door sensors, etc.
