# 03 — Network Configuration

Configure your managed switch with VLANs to separate Proxmox cluster traffic from regular network traffic, then configure the corresponding network interfaces on each Proxmox node.

---

## Why This Matters

Proxmox clustering uses a protocol called **Corosync** to keep all 3 nodes in sync — tracking which VMs are running where, health status, and cluster quorum. Corosync generates constant low-latency traffic between nodes.

Putting this traffic on a **dedicated VLAN** (separate from your regular home network and VM traffic) prevents it from competing with other traffic and ensures the cluster stays healthy. A managed switch lets you create VLANs — an unmanaged switch cannot.

---

## What You Need

- Access to your managed switch's admin interface (usually via browser at its IP, e.g., `192.168.1.2`)
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

## Part 1: Managed Switch Configuration

The exact interface varies by switch brand, but the concepts are universal. Look for these settings in your switch admin panel.

### Step 1.1: Identify the Ports

Note which physical ports on your switch each HP EliteDesk is plugged into. For example:
- Port 1 → pve1
- Port 2 → pve2
- Port 3 → pve3
- Port 8 → Your home router/uplink

### Step 1.2: Create VLAN 10

1. Navigate to **VLANs** or **802.1Q VLAN** settings
2. Create a new VLAN with ID **10** and name it **proxmox-cluster**
3. Add ports 1, 2, and 3 (the node ports) as **Tagged** members of VLAN 10
4. Do **not** add your router/uplink port to VLAN 10 — cluster traffic should never leave your local switch

> **Tagged vs Untagged**: Tagged means the port sends/receives frames with VLAN 10 labels (multiple VLANs on one cable). Untagged means the port uses that VLAN without labels (one VLAN per cable). Your nodes' ports should be **tagged for VLAN 10** and **untagged (native) for VLAN 1**.

### Step 1.3: Verify VLAN 1 (Management)

VLAN 1 is typically the default untagged VLAN. Ensure:
- Ports 1, 2, 3 have VLAN 1 as their **PVID** (Port VLAN ID / native VLAN)
- Your router/uplink port is also on VLAN 1

### Step 1.4: Save Switch Configuration

Apply and save changes on the switch.

---

## Part 2: Configure Proxmox Network Interfaces

Now configure each node to use VLAN-tagged interfaces. Do this on **all 3 nodes** via the Proxmox web UI or shell.

### Step 2.1: Open Network Configuration

In the Proxmox web UI:
1. Click on your node (e.g., `pve1`) in the left panel
2. Go to **System** → **Network**

You'll see the current interfaces:
- `eno1` — the physical Ethernet port
- `vmbr0` — the Linux bridge created during install (for VMs)

### Step 2.2: Create the VLAN 10 Interface

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

### Step 2.3: Verify vmbr0 (Management Bridge)

Check that `vmbr0` is correctly configured:
- Bridge ports: `eno1`
- IP Address: your node's management IP (e.g., `192.168.1.11`)
- Gateway: `192.168.1.254`

This bridge is what VMs use for network access. It passes VLAN 1 (untagged) traffic.

### Step 2.4: Apply Changes

Click **Apply Configuration** at the top of the Network page.

> Proxmox may warn that applying network changes can disconnect you — that's okay. If you lose access, the machine will still be reachable at its management IP after a few seconds.

### Step 2.5: Repeat for pve2 and pve3

On pve2: create `eno1.10` with IP `10.10.10.2`
On pve3: create `eno1.10` with IP `10.10.10.3`

---

## Part 3: Verify Connectivity

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
- Double-check the switch VLAN tagging on all 3 ports
- Confirm the VLAN sub-interface IPs are correct on each node
- Make sure the switch VLAN 10 does **not** have a gateway configured (it's an isolated VLAN)

Also verify management IPs still work:
```bash
ping 192.168.1.12 -c 4   # from pve1, should reach pve2
ping 192.168.1.13 -c 4   # from pve1, should reach pve3
```

---

## Understanding the Final Network Layout

```
              eno1 (physical port)
                │
       ┌────────┴────────┐
       │                 │
  vmbr0 (bridge)    eno1.10 (VLAN 10)
  192.168.1.11      10.10.10.1
       │
  (VM traffic + web UI)   (corosync cluster heartbeat only)
```

- `vmbr0` is a Linux bridge that your VMs connect to — think of it like a virtual switch inside the machine
- `eno1.10` is a VLAN-tagged sub-interface that only carries traffic tagged with VLAN ID 10
- Corosync (the cluster protocol) will be configured to use the `10.10.10.x` addresses in the next guide

---

## Checklist

- [ ] VLAN 10 created on managed switch
- [ ] Node ports tagged for VLAN 10 on the switch
- [ ] `eno1.10` VLAN interface created on pve1 (IP: 10.10.10.1)
- [ ] `eno1.10` VLAN interface created on pve2 (IP: 10.10.10.2)
- [ ] `eno1.10` VLAN interface created on pve3 (IP: 10.10.10.3)
- [ ] Cross-node pings on 10.10.10.x all succeed
- [ ] Management IPs (192.168.1.x) still reachable

---

## Next Step

Proceed to [04 - Cluster Setup](04-cluster-setup.md) to create the Proxmox cluster using these network interfaces.
