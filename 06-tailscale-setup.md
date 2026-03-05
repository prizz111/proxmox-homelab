# 06 — Tailscale Setup

Install Tailscale on all 3 Proxmox nodes to enable secure remote access to your homelab from anywhere.

---

## Why This Matters

By default, your Proxmox web UI (port 8006) is only accessible from your home network. To reach it remotely, you'd normally need to open firewall ports or set up a VPN server.

**Tailscale** solves this without opening any ports. It creates a **peer-to-peer encrypted mesh network** (based on WireGuard) between your devices. Your Proxmox nodes join your personal Tailscale network and become reachable from your phone, laptop, or any other Tailscale-connected device — securely, even on public WiFi.

Each device gets a stable `100.x.x.x` IP that works from anywhere.

---

## Prerequisites

- Free Tailscale account — create one at [tailscale.com](https://tailscale.com) before starting
- All 3 Proxmox nodes online with internet access
- Tailscale client installed on your phone or laptop

---

## Step 1: Install Tailscale on Each Node

On pve1 (repeat for pve2 and pve3):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Tailscale will output an authentication URL. Open it in your browser and log in with your Tailscale account.

Check the status:

```bash
tailscale status
```

After all 3 nodes are connected, all 3 should appear with `active; direct` status — meaning they're talking directly over WireGuard tunnels.

---

## Step 2: Enable Tailscale SSH (Optional but Recommended)

Enable it on each node:

```bash
tailscale up --ssh
```

Now from any Tailscale-connected device:

```bash
ssh root@pve1
```

Tailscale resolves the hostname automatically using its built-in MagicDNS feature.

---

## Step 3: Access Proxmox Web UI Remotely

From any device on your Tailscale network:

1. Find pve1's Tailscale IP from the [Tailscale admin console](https://login.tailscale.com/admin/machines)
2. Open `https://100.x.x.x:8006` in your browser
3. Proceed past the SSL warning and log in normally

---

## Step 4: Enable MagicDNS

1. Go to the [Tailscale admin console](https://login.tailscale.com/admin/dns)
2. Under **DNS**, enable **MagicDNS**

Now you can use `https://pve1:8006` instead of `https://100.x.x.x:8006`.

---

## Step 5: Install Tailscale on Personal Devices

- **Phone**: App Store or Google Play → sign in
- **Laptop**: [tailscale.com/download](https://tailscale.com/download) → install → sign in

---

## Optional: Subnet Routing

To reach **other home network devices** (not just the 3 nodes) via Tailscale:

```bash
tailscale up --ssh --advertise-routes=192.168.1.0/24
```

Then approve the subnet route in the Tailscale admin console.

---

## Checklist

- [ ] Tailscale installed and authenticated on pve1
- [ ] Tailscale installed and authenticated on pve2
- [ ] Tailscale installed and authenticated on pve3
- [ ] `tailscale status` shows all 3 nodes with `active; direct` connections
- [ ] Tailscale SSH enabled on each node
- [ ] MagicDNS enabled in Tailscale admin console
- [ ] Proxmox web UI accessible via Tailscale from an external device
- [ ] Tailscale app installed on phone/laptop

---

## Next Step

Proceed to [07 - Immich Setup](07-immich-setup.md) to deploy your self-hosted photo management server.
