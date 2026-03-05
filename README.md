# Proxmox Home Cluster

A complete setup guide for building a 3-node Proxmox VE cluster using HP EliteDesk 800 G5 Desktop Mini machines, with Tailscale for secure remote access and Immich for self-hosted photo management.

## Why This Project

This build serves two goals:
1. **Learning**: Hands-on experience with home networking, virtualization, clustering, and self-hosted services
2. **Practical use**: A capable, low-power home server cluster to run real applications

The HP EliteDesk 800 G5 Desktop Mini is an ideal choice — compact, quiet, power-efficient (~35W under load), and capable enough to run multiple VMs and containers simultaneously.

---

## Hardware Inventory

| Component | Details |
|---|---|
| Nodes | 3× HP EliteDesk 800 G5 Desktop Mini |
| CPU | Intel Core i5 9th Gen (per node) |
| RAM | 16GB DDR4 (per node) |
| OS Drive | M.2 NVMe SSD (existing, per node) |
| Data Drive | 2.5" SATA HDD (purchased separately, per node) |
| Network | Intel I219-LM 1GbE (per node) |
| Switch | UniFi Flex Mini (managed, VLAN-capable) |

---

## Architecture Overview

```
                        ┌──────────────────────────────────────────┐
                        │           Tailscale Mesh Network          │
                        │         (WireGuard overlay, 100.x.x.x)   │
                        └───────────────┬──────────────────────────┘
                                        │ secure tunnel
                                        │
                              ┌─────────▼──────────┐
                              │  AT&T BGW320 Gateway │
                              └─────────┬──────────┘
                                        │
                              ┌─────────▼──────────┐
                              │  UniFi Flex Mini    │
                              │  VLAN 1 (mgmt/VMs) │
                              │  VLAN 10 (cluster) │
                              └──┬──────┬──────┬───┘
                                 │      │      │
                     ┌───────────┘  ┌───┘  └───────────┐
                     │              │                   │
              ┌──────▼─────┐ ┌──────▼─────┐ ┌──────────▼──┐
              │    pve1    │ │    pve2    │ │    pve3     │
              │192.168.1.101│ │192.168.1.102│ │192.168.1.103│
              │            │ │            │ │             │
              │  [Immich]  │ │            │ │             │
              │  VM        │ │            │ │             │
              └──────┬─────┘ └────────────┘ └─────────────┘
                     │
              ┌──────▼─────┐
              │  SATA HDD  │  (local data storage, per node)
              └────────────┘
```

**Key concepts:**
- **Proxmox VE** is a Type 1 hypervisor — it runs directly on bare metal, not inside another OS
- **Clustering** ties all 3 nodes into a single management interface; VMs can be live-migrated between nodes
- **Quorum**: 3 nodes means the cluster stays operational if 1 node fails (majority vote)
- **Tailscale** creates an encrypted mesh network so you can reach your homelab from anywhere without opening firewall ports
- **Local storage**: Each node has its own SATA HDD — simpler than distributed storage (Ceph) and great for learning

---

## Network Address Plan

| Host | Hostname | Management IP | Cluster IP | Tailscale IP |
|---|---|---|---|---|
| Node 1 | pve1.homelab.local | 192.168.1.101 | 10.10.10.1 | assigned by Tailscale |
| Node 2 | pve2.homelab.local | 192.168.1.102 | 10.10.10.2 | assigned by Tailscale |
| Node 3 | pve3.homelab.local | 192.168.1.103 | 10.10.10.3 | assigned by Tailscale |
| Immich VM | immich.homelab.local | 192.168.1.110 | — | assigned by Tailscale |

> **Note**: Adjust `192.168.1.x` to match your router's subnet if needed.

---

## Reading Order

Follow these guides in sequence:

1. [01 - Hardware Preparation](01-hardware-prep.md) — Install HDDs, configure BIOS
2. [02 - Proxmox Installation](02-proxmox-install.md) — Install Proxmox VE 8 on each node
3. [03.1 - Switch Setup](03.1-switch-setup.md) — First-time UniFi Flex Mini setup with AT&T
4. [03 - Network Configuration](03-network-config.md) — Configure VLANs on Proxmox nodes
5. [04 - Cluster Setup](04-cluster-setup.md) — Form the 3-node Proxmox cluster
6. [05 - Storage Setup](05-storage-setup.md) — Add SATA HDDs as local storage
7. [06 - Tailscale Setup](06-tailscale-setup.md) — Secure remote access via Tailscale
8. [07 - Immich Setup](07-immich-setup.md) — Self-hosted photo management

---

## Time Estimate

| Guide | Estimated Time |
|---|---|
| Hardware Prep (all 3 nodes) | 45–60 min |
| Proxmox Installation (all 3 nodes) | 60–90 min |
| Switch Setup | 30–45 min |
| Network Configuration | 20–30 min |
| Cluster Setup | 20–30 min |
| Storage Setup | 20–30 min |
| Tailscale Setup | 15–20 min |
| Immich Setup | 30–45 min |
| **Total** | **~4–5 hours** |

---

## Prerequisites

- USB drive (8GB+) for the Proxmox installer
- A Windows or Linux computer to create the bootable USB
- Internet connection during setup (for package downloads)
- A free Ubiquiti account (create at account.ui.com before starting guide 03.1)
- A free Tailscale account (create at tailscale.com before starting guide 06)
