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

**Expected output:**
```
Cluster information
-------------------
Name:             homelab-cluster
Config Version:   1
Transport:        knet
Secure auth:      on

Quorum information
------------------
Date:             ...
Quorum provider:  corosync_votequorum
Nodes:            1
Node ID:          0x00000001
Ring ID:          1.8
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   1
Highest expected: 1
Total votes:      1
Quorum:           1
Flags:            Quorate

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 pve1 (local)
```

With only 1 node, `Expected votes: 1` and `Quorate: Yes` is correct.

---

## Step 2: Join pve2 to the Cluster

### Option A: Via Web UI (Easiest)

1. Open the Proxmox web UI for pve2: `https://192.168.1.12:8006`
2. Go to **Datacenter** → **Cluster** → **Join Cluster**
3. On pve1, go to **Datacenter** → **Cluster** → **Join Information**
4. Copy the join information from pve1 and paste it into the pve2 join dialog
5. Enter pve1's root password when prompted
6. In the **Cluster Network** field, change the IP to `10.10.10.2` (pve2's VLAN 10 IP)
7. Click **Join**

### Option B: Via Command Line

SSH into pve2, then run:

```bash
pvecm add 192.168.1.11 --link0 10.10.10.2
```

- `192.168.1.11` — management IP of pve1 (used to fetch cluster config)
- `--link0 10.10.10.2` — pve2's VLAN 10 IP for cluster traffic

You'll be prompted for pve1's root password. Enter it.

**Wait about 30 seconds** for cluster sync to complete, then verify:

```bash
pvecm status
```

You should now see `Nodes: 2` and both nodes in the membership list.

---

## Step 3: Join pve3 to the Cluster

SSH into pve3, then run:

```bash
pvecm add 192.168.1.11 --link0 10.10.10.3
```

Enter pve1's root password when prompted.

Wait 30 seconds, then verify from any node:

```bash
pvecm status
```

**Expected final output:**
```
Cluster information
-------------------
Name:             homelab-cluster
Config Version:   3
Transport:        knet
Secure auth:      on

Quorum information
------------------
Nodes:            3
Quorate:          Yes

Votequorum information
----------------------
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2

Membership information
----------------------
    Nodeid      Votes Name
0x00000001          1 pve1 (local)
0x00000002          1 pve2
0x00000003          1 pve3
```

Key things to confirm:
- `Nodes: 3`
- `Quorate: Yes`
- `Expected votes: 3`
- All 3 nodes listed in Membership

---

## Step 4: Verify in the Web UI

Open `https://192.168.1.11:8006` (pve1's web UI — but now it's the cluster UI).

In the left panel under **Datacenter**, you should see all 3 nodes:
```
Datacenter (homelab-cluster)
├── pve1
├── pve2
└── pve3
```

Clicking on each node shows its individual resources. Clicking on **Datacenter** at the top shows cluster-wide views.

> **Note**: After joining, you only need to open one node's web UI — all 3 nodes are visible and manageable from any single node's interface.

---

## Step 5: Verify Node Health

Run on any node:

```bash
pvecm nodes
```

```
Membership information
----------------------
    Nodeid      Votes Quorum_votes Name
         1          1            1 pve1 (local)
         2          1            0 pve2
         3          1            0 pve3
```

Also check no errors:

```bash
systemctl status corosync
systemctl status pve-cluster
```

Both should show `active (running)`.

---

## Understanding Cluster Storage of Config

Once clustered, `/etc/pve/` becomes a distributed filesystem shared across all nodes (using PMXCFS, the Proxmox cluster filesystem). This means:
- VM configurations created on any node are visible on all nodes
- Cluster-wide users, permissions, and storage definitions are in sync automatically
- If a node goes offline, its config is still readable from the surviving nodes

---

## Troubleshooting

**Node won't join — "corosync not running"**
```bash
systemctl start corosync
systemctl status corosync
```

**Quorum lost (only 1 or 2 nodes up after testing)**

If you deliberately shut down nodes for testing and the cluster loses quorum:
```bash
pvecm expected 1   # temporarily override quorum to 1 node
```
This is for maintenance only — **undo it** before bringing nodes back:
```bash
pvecm expected 3
```

**Wrong link IP used during join**

You can update the corosync link config in `/etc/pve/corosync.conf` — but be careful, as cluster config is shared. Ask on the [Proxmox forum](https://forum.proxmox.com) if unsure.

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
