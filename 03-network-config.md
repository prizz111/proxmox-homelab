# 03 — Network Configuration

Configure the VLAN interfaces on each Proxmox node to match the switch configuration from guide 03.1.

---

## Why This Matters

Proxmox clustering uses a protocol called **Corosync** to keep all 3 nodes in sync — tracking which VMs are running where, health status, and cluster quorum. Corosync generates constant low-latency traffic between nodes.

Putting this traffic on a **dedicated VLAN** (separate from your regular home network and VM traffic) prevents it from competing with other traffic and ensures the cluster stays healthy.

---

## What You Need

- Switch configured per [03.1 - Switch Setup](03.1-switch-setup.md) (VLAN 10 and `proxmox-trunk` port profile applied)
- Each Proxmox node accessible via SSH or web shell

---

## Network Design

We'll use two VLANs:

| VLAN | ID | Purpose | Subnet |
|---|---|---|---|
| Management + VMs | 1 (untagged/native) | Proxmox web UI, VM traffic, general home network | 192.168.1.0/24 |
| Cluster (Corosync) | 10 | Internal Proxmox cluster heartbeat traffic only | 10.10.10.0/24 |

The 3 Proxmox nodes will have **one physical Ethernet port** each (the Intel I219-LM on the HP EliteDesk). Since there's only one port, we'll use **VLAN tagging** (802.1Q) to carry both VLANs over that single cable.

---

## Configure Proxmox Network Interfaces

Do this on **all 3 nodes** via the Proxmox web UI or shell.

### Step 1: Open Network Configuration

In the Proxmox web UI:
1. Click on your node (e.g., `pve1`) in the left panel
2. Go to **System** → **Network**

You'll see the current interfaces:
- `eno1` — the physical Ethernet port
- `vmbr0` — the Linux bridge created during install (for VMs)

### Step 2: Create the VLAN 10 Interface

We'll create a VLAN sub-interface on `eno1` for cluster traffic.

Click **Create** → **Linux VLAN**:
| Field | Value |
|---|---|
| Name | `eno1.10` |
| VLAN Raw Device | `eno1` |
| VLAN Tag | `10` |
| IP Address | `10.10.10.1` (use `.2` for pve2, `.3` for pve3) |
| Subnet mask | `255.255.255.0` (/24) |
| Gateway | *(leave empty — only used for cluster traffic)* |
| Comment | `Proxmox cluster/corosync` |

Click **Create**.

### Step 3: Verify vmbr0 (Management Bridge)

Check that `vmbr0` is correctly configured:
- Bridge ports: `eno1`
- IP Address: your node's management IP (e.g., `192.168.1.101`)
- Gateway: `192.168.1.1`

This bridge is what VMs use for network access. It passes VLAN 1 (untagged) traffic.

### Step 4: Apply Changes

Click **Apply Configuration** at the top of the Network page.

> Proxmox may warn that applying network changes can disconnect you — that's okay. If you lose access, the machine will still be reachable at its management IP after a few seconds.

### Step 5: Repeat for pve2 and pve3

On pve2: create `eno1.10` with IP `10.10.10.2`
On pve3: create `eno1.10` with IP `10.10.10.3`

---

## Verify Connectivity

From the shell of pve1, ping the cluster IPs of pve2 and pve3:

```bash
ping 10.10.10.2 -c 4
ping 10.10.10.3 -c 4
```

Expected output:
```
64 bytes from 10.10.10.2: icmp_seq=1 ttl=64 time=0.4 ms
64 bytes from 10.10.10.2: icmp_seq=2 ttl=64 time=0.3 ms
...
```

If pings fail:
- Double-check the switch `proxmox-trunk` profile is applied to ports 1, 2, 3 in UniFi
- Confirm the VLAN sub-interface IPs are correct on each node
- Make sure the switch VLAN 10 does **not** have a gateway configured (it's an isolated VLAN)

Also verify management IPs still work:
```bash
ping 192.168.1.102 -c 4   # from pve1, should reach pve2
ping 192.168.1.103 -c 4   # from pve1, should reach pve3
```

---

## Understanding the Final Network Layout

```
              eno1 (physical port)
                │
       ┌────────┴────────┐
       │                 │
  vmbr0 (bridge)    eno1.10 (VLAN 10)
  192.168.1.101     10.10.10.1
       │
  (VM traffic + web UI)   (corosync cluster heartbeat only)
```

- `vmbr0` is a Linux bridge that your VMs connect to — think of it like a virtual switch inside the machine
- `eno1.10` is a VLAN-tagged sub-interface that only carries traffic tagged with VLAN ID 10
- Corosync (the cluster protocol) will be configured to use the `10.10.10.x` addresses in the next guide

---

## Checklist

- [ ] `eno1.10` VLAN interface created on pve1 (IP: 10.10.10.1)
- [ ] `eno1.10` VLAN interface created on pve2 (IP: 10.10.10.2)
- [ ] `eno1.10` VLAN interface created on pve3 (IP: 10.10.10.3)
- [ ] Cross-node pings on 10.10.10.x all succeed
- [ ] Management IPs (192.168.1.x) still reachable

---

## Next Step

Proceed to [04 - Cluster Setup](04-cluster-setup.md) to create the Proxmox cluster using these network interfaces.
