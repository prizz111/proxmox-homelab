# 05 — Storage Setup

Format the 2.5" SATA HDD on each node and add it as local storage in Proxmox.

**Repeat this process on each of the 3 nodes. Each node's SATA HDD is independent — it does not sync with the others.**

---

## Why This Matters

Proxmox installed onto your NVMe drive and created two default storage pools:
- **`local`** — stores ISOs, backups, and container templates (on the NVMe)
- **`local-lvm`** — stores VM disk images using LVM (on the NVMe)

Your NVMe is the OS drive — it's not where you want large VM disks or media files living. The SATA HDD gives each node dedicated bulk storage, keeping the OS drive clean and extending your overall capacity.

We'll format the SATA HDD with `ext4` and add it to Proxmox as a **Directory** storage type — the simplest option for a homelab.

---

## Step 1: Identify the SATA HDD

On each node, open the shell (Proxmox web UI → node → Shell) and run:

```bash
lsblk
```

In the output:
- `nvme0n1` = your NVMe OS drive (has Proxmox on it, already partitioned)
- `sda` = your SATA HDD (no partitions listed — it's blank)

> If you're unsure which is which, run `fdisk -l /dev/sda` to see size and model info.

**Make note of the device name** (likely `sda` but could be `sdb`).

---

## Step 2: Partition the SATA HDD

> **Warning**: This erases everything on the SATA HDD. Since it's new, that's fine.

```bash
fdisk /dev/sda
```

Type these commands one at a time inside fdisk:

```
g       # Create a new GPT partition table
n       # Add a new partition (press Enter 3 times for defaults)
w       # Write changes and exit
```

Verify with `lsblk /dev/sda` — you should now see `sda1`.

---

## Step 3: Format the Partition

```bash
mkfs.ext4 /dev/sda1
```

---

## Step 4: Create the Mount Point and Mount the Drive

```bash
mkdir -p /mnt/pve/local-sata
```

Get the UUID:

```bash
blkid /dev/sda1
# Example: /dev/sda1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4"
```

Add to fstab (replace UUID with your actual value):

```bash
echo 'UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890 /mnt/pve/local-sata ext4 defaults 0 2' >> /etc/fstab
mount -a
```

Verify:

```bash
df -h /mnt/pve/local-sata
```

---

## Step 5: Add the Storage to Proxmox

### Via Web UI (Recommended)

1. **Datacenter** → **Storage** → **Add** → **Directory**
2. Fill in the form:

| Field | Value for pve1 | Notes |
|---|---|---|
| ID | `local-sata-pve1` | Unique name — change the number per node |
| Directory | `/mnt/pve/local-sata` | The mount point you created |
| Content | Disk image, Container, ISO image, Snippets | Select all |
| Nodes | `pve1` | **Important**: restrict to this node only |

3. Click **Add**

### Via Command Line

```bash
pvesm add dir local-sata-pve1 --path /mnt/pve/local-sata \
  --content images,rootdir,iso,snippets \
  --nodes pve1
```

---

## Step 6: Repeat for pve2 and pve3

| Node | Storage ID |
|---|---|
| pve1 | `local-sata-pve1` |
| pve2 | `local-sata-pve2` |
| pve3 | `local-sata-pve3` |

---

## Checklist

- [ ] SATA HDD identified as `sda` (or `sdb`) on each node
- [ ] Partitioned with GPT using `fdisk`
- [ ] Formatted with `ext4`
- [ ] Mount point created at `/mnt/pve/local-sata`
- [ ] UUID added to `/etc/fstab`
- [ ] Drive verified mounted with `df -h`
- [ ] Proxmox storage added via web UI or `pvesm` per node
- [ ] All 3 `local-sata-pveX` storage entries show **Active** in Datacenter → Storage

---

## Next Step

Proceed to [06 - Tailscale Setup](06-tailscale-setup.md) to set up secure remote access to your cluster.
