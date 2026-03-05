# 01 — Hardware Preparation

Prepare all 3 HP EliteDesk 800 G5 Desktop Mini machines before installing Proxmox. This involves installing the SATA HDD internally and configuring BIOS settings for virtualization.

**Repeat every step in this guide for all 3 nodes.**

---

## Why This Matters

The HP EliteDesk 800 G5 Mini's BIOS ships with virtualization disabled by default — Proxmox requires Intel VT-x and VT-d to be enabled or VMs won't run. The 2.5" SATA bay sits next to the M.2 slot inside the chassis; installing it now (before the OS) means Proxmox will detect it during setup.

---

## Tools Needed

- Phillips #1 screwdriver
- Anti-static wrist strap (recommended)
- Your 2.5" SATA HDD

---

## Step 1: Open the Chassis

The HP EliteDesk 800 G5 Mini uses a tool-free side panel.

1. Power off the machine and unplug all cables
2. Lay the machine flat with the side panel facing up
3. Locate the **green thumbscrew** on the back panel — turn it counter-clockwise until it releases
4. Slide the top panel toward the rear of the machine, then lift it off
5. You now have access to the internals

> The M.2 NVMe (your future Proxmox OS drive) is typically already installed here. Do not remove it.

---

## Step 2: Install the 2.5" SATA HDD

The 2.5" SATA bay is located near the front of the chassis, just above the M.2 slot area.

1. Locate the **2.5" drive bracket** — it's a metal carrier that slides out from the front
2. Gently pull the bracket toward you to slide it out (it may have a green tab or release lever)
3. Place your 2.5" SATA HDD into the bracket with the SATA connector facing the back of the bracket
4. Secure the drive with the 4 small screws on the sides of the bracket (usually included with the machine or the drive)
5. Slide the bracket back in until it clicks and the SATA connector seats into the onboard SATA port
6. Close the top panel and tighten the thumbscrew

> **Tip**: The SATA connection is a direct-connect design — there is no separate SATA cable. The drive connects directly to a port on the bracket rail.

---

## Step 3: Configure BIOS

Power the machine on and immediately press **F10** repeatedly to enter BIOS Setup (HP uses F10 for BIOS, not F2 or DEL).

### 3a. Enable Intel Virtualization Technology (VT-x)

Required for running virtual machines.

1. Go to **Advanced** → **System Options**
2. Find **Virtualization Technology (VTx)** — set to **Enabled**

### 3b. Enable Intel VT-d (IOMMU)

Required for PCI passthrough (e.g., GPU or USB controller passthrough to VMs). Good to enable now even if not immediately needed.

1. Still in **Advanced** → **System Options**
2. Find **Virtualization Technology for Directed I/O (VTd)** — set to **Enabled**

### 3c. Set Boot Order (USB First)

You'll need to boot from a USB drive to install Proxmox.

1. Go to **Advanced** → **Boot Options**
2. Find **UEFI Boot Order** or **Boot Priority**
3. Move **USB Drive** to the top of the list
4. Also ensure **Legacy Boot** is enabled or set to **UEFI + Legacy** if your USB was created in legacy mode

> **Note**: Proxmox installs in UEFI mode by default when Secure Boot is off. Keep the boot mode as **UEFI**.

### 3d. Disable Secure Boot

Proxmox's bootloader is not Secure Boot signed.

1. Go to **Security** → **Secure Boot Configuration** (or **Security** → **Secure Boot**)
2. Set **Secure Boot** to **Disabled**

### 3e. Enable Wake-on-LAN (Optional but Recommended)

Allows you to power on nodes remotely from your network.

1. Go to **Advanced** → **Built-In Device Options**
2. Find **S5 Wake on LAN** — set to **Enabled**
3. Also check **Remote Wake Up Boot Source** — set to **Remote Server** or **Local Hard Drive** (either works)

### 3f. Save and Exit

Press **F10** or navigate to **Exit** → **Save Changes and Exit**. The machine will reboot.

---

## Step 4: Verify HDD Detection

Before installing Proxmox, confirm the BIOS sees both drives.

1. Press **F10** again to re-enter BIOS
2. Go to **Main** or **Storage** — you should see:
   - Your M.2 NVMe listed (the OS drive)
   - Your new SATA HDD listed (the data drive)

If the SATA HDD doesn't appear:
- Power off and reseat the drive bracket
- Make sure the bracket clicked fully into place — the SATA connector must be fully engaged

---

## Checklist (Per Node)

- [ ] SATA HDD physically installed in 2.5" bay
- [ ] VT-x enabled in BIOS
- [ ] VT-d enabled in BIOS
- [ ] USB boot priority set first
- [ ] Secure Boot disabled
- [ ] Wake-on-LAN enabled (optional)
- [ ] Both drives (NVMe + SATA) visible in BIOS storage list

---

## Next Step

Once all 3 nodes have their HDDs installed and BIOS configured, proceed to [02 - Proxmox Installation](02-proxmox-install.md).
