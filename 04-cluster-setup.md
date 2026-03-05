# 04 — Proxmox Cluster Setup

Create a 3-node Proxmox cluster that ties all your machines into a single unified management interface.

---

## Why This Matters

Right now, each Proxmox node is an independent machine managed separately. A **Proxmox cluster** joins them so that:

- All 3 nodes appear in a single web UI
- You can move VMs between nodes (live migration)
- The cluster tracks health and quorum — if 1 node goes down, the other 2 can still operate
- Cluster-wide settings (users, storage, SDN) are shared automatically

**Quorum explained**: Proxmox uses a voting system. With 3 nodes, you need at least 2 to agree on cluster state. If 1 node fails, the other 2 have a majority (2 of 3 = quorum maintained). This prevents "split brain" — where 2 isolated halves of a cluster both think they're the primary.

> **Order matters**: Create the cluster on pve1 first, then join pve2 and pve3 one at a time.

---

## Prerequisites

- All 3 nodes are installed and accessible at their management IPs
- Network is configured (VLAN 10 / 10.10.10.x is reachable between nodes)
- All nodes have the same root password (required for SSH-based joining)
- All nodes are running the same Proxmox VE version (`pveversion -v` to check)

---

## Step 1: Create the Cluster on pve1

SSH into pve1 or use the Proxmox web UI shell (pve1 → Shell):

```bash
pvecm create homelab-cluster --link0 10.10.10.1
```

Breaking down this command:
- `pvecm create` — creates a new cluster
- `homelab-cluster` — the cluster name (can be anything, used for identification)
- `--link0 10.10.10.1` — tells Corosync to use the VLAN 10 interface (10.10.10.1) for cluster traffic, not the management IP

**Expected output:**
```
Corosync Cluster Engine Authentication key generator.
Gathering 2048 bits for key from /dev/urandom.
Writing corosync key to /etc/corosync/authkey.
Writing configuration to /etc/pve/corosync.conf
...
```

Verify the cluster is running:

```bash
pvecm status
```

With only 1 node, `Expected votes: 1` and `Quorate: Yes` is correct.

---

## Step 2: Join pve2 to the Cluster

### Option A: Via Web UI (Easiest)

1. Open the Proxmox web UI for pve2: `https://192.168.1.102:8006`
2. Go to **Datacenter** → **Cluster** → **Join Cluster**
3. On pve1, go to **Datacenter** → **Cluster** → **Join Information**
4. Copy the join information from pve1 and paste it into the pve2 join dialog
5. Enter pve1's root password when prompted
6. In the **Cluster Network** field, change the IP to `10.10.10.2` (pve2's VLAN 10 IP)
7. Click **Join**

### Option B: Via Command Line

SSH into pve2, then run:

```bash
pvecm add 192.168.1.101 --link0 10.10.10.2
```

You'll be prompted for pve1's root password. Enter it.

**Wait about 30 seconds** for cluster sync to complete, then verify:

```bash
pvecm status
```

---

## Step 3: Join pve3 to the Cluster

SSH into pve3, then run:

```bash
pvecm add 192.168.1.101 --link0 10.10.10.3
```

Wait 30 seconds, then verify from any node:

```bash
pvecm status
```

**Expected final output confirms:**
- `Nodes: 3`
- `Quorate: Yes`
- `Expected votes: 3`
- All 3 nodes listed in Membership

---

## Step 4: Verify in the Web UI

Open `https://192.168.1.101:8006`. In the left panel under **Datacenter**, you should see all 3 nodes:
```
Datacenter (homelab-cluster)
├── pve1
├── pve2
└── pve3
```

> **Note**: After joining, you only need to open one node's web UI — all 3 nodes are visible and manageable from any single node's interface.

---

## Step 5: Verify Node Health

```bash
pvecm nodes
systemctl status corosync
systemctl status pve-cluster
```

Both services should show `active (running)`.

---

## Troubleshooting

**Node won't join — "corosync not running"**
```bash
systemctl start corosync
systemctl status corosync
```

**Quorum lost (only 1 or 2 nodes up after testing)**

```bash
pvecm expected 1   # temporarily override quorum to 1 node
# undo before bringing nodes back:
pvecm expected 3
```

---

## Checklist

- [ ] Cluster created on pve1 with `--link0 10.10.10.1`
- [ ] pve2 joined with `--link0 10.10.10.2`
- [ ] pve3 joined with `--link0 10.10.10.3`
- [ ] `pvecm status` shows 3 nodes, `Quorate: Yes`, `Expected votes: 3`
- [ ] All 3 nodes visible in Proxmox web UI under Datacenter
- [ ] `systemctl status corosync` shows `active (running)` on all nodes

---

## Next Step

Proceed to [05 - Storage Setup](05-storage-setup.md) to format and add the SATA HDDs as local storage on each node.
