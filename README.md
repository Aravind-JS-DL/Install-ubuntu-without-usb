# ubuntu-dualboot-no-usb

> **Install Ubuntu alongside Windows 10/11 — no USB drive, no CD, no external hardware.**  
> A modern, verified guide for UEFI/GPT systems using the "Sleight of Hand" method.

---

## Table of Contents

- [Disclaimer](#️-disclaimer)
- [Why This Guide Exists](#why-does-this-guide-exist)
- [How It Works](#the-sleight-of-hand--how-it-works)
- [Prerequisites](#prerequisites)
- [Phase 1 — Partition Your Drive in Windows](#phase-1--partition-your-drive-in-windows)
- [Phase 2 — Set the UEFI Boot Flag](#phase-2--set-the-uefi-boot-flag)
- [Phase 3 — The Sleight of Hand](#phase-3--the-sleight-of-hand-live-environment)
- [Phase 4 — Manual Installation](#phase-4--manual-installation)
- [Phase 5 — Cleanup](#phase-5--cleanup)

---

## ⚠️ Disclaimer

This guide involves **modifying disk partitions and bootloader configurations**. If you skip steps, misread partition identifiers, or deviate from the instructions, you risk data loss or an unbootable system.

- Follow every step carefully and **in order**.
- Always double-check partition names (e.g. `/dev/sda3`) before running any command — selecting the wrong partition can be **irreversible**.
- **Back up your important files before you begin. No exceptions.**

> This guide is provided as-is. The author is **not responsible** for any damage, data loss, or system failures resulting from misuse, skipped steps, or mistakes made during the process. You are solely responsible for your own system.

---

## Why Does This Guide Exist?

Every existing "install Ubuntu without a USB" guide is broken in at least one of these ways:

| Method | Why It Fails |
|---|---|
| UNetbootin / EasyBCD | Built for legacy BIOS/MBR. Obsolete and blocked on modern UEFI systems. |
| Extract ISO → boot from internal partition | The Ubuntu installer flags that partition as an active system drive, locks the disk, and crashes GRUB with a `/cow` or `/boot/efi` error. |
| `toram` + force-unmount workaround | Modern Ubuntu uses **Subiquity** as its installer backend. Unmounting the drive causes Subiquity to lose its `/cdrom/cloud-init` config files, crashing the installer in an infinite loop. |

This guide solves all three problems without any external hardware.

---

## The "Sleight of Hand" — How It Works

1. A temporary **8 GB partition** is assigned the official EFI boot flag in Windows — tricking your motherboard into booting the Ubuntu installer directly from your internal drive.
2. Once the installer loads, **the flag is stripped away** while the system is still running.
3. The installer files remain actively mounted (keeping Subiquity happy), but the fake EFI partition is now invisible to the installer — allowing a clean, error-free GRUB installation onto your real system drive.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **System** | Windows 10 or 11 PC with a modern UEFI motherboard |
| **Ubuntu ISO** | Download from [ubuntu.com/download/desktop](https://ubuntu.com/download/desktop) — verified on `ubuntu-24.04.4-desktop-amd64`, compatible with any current release |
| **Installer space** | ~8 GB temporary partition (fully reclaimed after installation) |
| **Ubuntu partition** | Minimum ~25 GB recommended — Ubuntu's base install is currently ~6 GB but grows with each release; allocate more if you plan to install apps or store files |
| **Backup** | Back up all critical files before modifying any partitions |

---

## Step-by-Step Guide

### Phase 1 — Partition Your Drive in Windows

> Partitioning in Windows first prevents the Ubuntu installer from rewriting the partition table mid-install, which can trigger disk locks.

1. Right-click the Start Menu → open **Disk Management**.

2. **Create the Installer partition:**
   - Right-click your `C:` or `D:` drive → **Shrink Volume** → enter `8192 MB` (8 GB).
   - Right-click the new Unallocated space → **New Simple Volume**.
   - Format as **FAT32** and label it `INSTALLER`.

3. **Create the Ubuntu target partition:**
   - Right-click your `C:` or `D:` drive again → **Shrink Volume**.
   - Enter at least `25600 MB` (25 GB). Allocate more if you can — Ubuntu benefits from the breathing room.
   - Leave this space as **Unallocated**. Do not format it.

4. **Copy the ISO contents:**
   - Double-click your downloaded Ubuntu `.iso` file. Windows will mount it as a virtual DVD drive.
   - Select **all** files and folders inside it.
   - Copy and paste them directly into the root of your `INSTALLER` drive.

---

### Phase 2 — Set the UEFI Boot Flag

> Modern UEFI motherboards only boot from a partition that carries a specific EFI GUID. We assign this flag temporarily to our installer partition.

1. Open **Command Prompt as Administrator**:  
   Start → type `cmd` → right-click → **Run as administrator**

2. Run:
   ```cmd
   diskpart
   list volume
   ```

3. Locate your `INSTALLER` volume in the list and note its **Volume #** (e.g. `4`).

4. Run the following commands, replacing `X` with your volume number:
   ```cmd
   select volume X
   set id=c12a7328-f81f-11d2-ba4b-00a0c93ec93b
   exit
   ```

> **What is that GUID?**  
> `c12a7328-f81f-11d2-ba4b-00a0c93ec93b` is the universal EFI System Partition identifier defined by the UEFI specification. Assigning it to a partition signals your motherboard to treat it as bootable — regardless of what's actually on it.

---

### Phase 3 — The Sleight of Hand (Live Environment)

> We boot the installer and immediately strip the fake EFI flag while it's running — before the installer gets a chance to use it.

1. Restart your PC and repeatedly press your BIOS boot menu key (`F12`, `F8`, `F2`, or `Del` — varies by manufacturer).

2. From the boot menu, select your **internal drive** — it may appear as a secondary **"UEFI OS"** entry.

3. Select **Try or Install Ubuntu**.  
   Modern Ubuntu ISOs will automatically launch the installer app in the background — this is expected behaviour.

4. **Before proceeding in the installer**, open Terminal:  
   `Ctrl` + `Alt` + `T`

5. Identify your `INSTALLER` partition:
   ```bash
   lsblk
   ```
   Look for your ~8 GB partition. Note the **drive name** (e.g. `sda`, `nvme0n1`) and **partition number** (e.g. `3`).

6. Strip the EFI flag (replace `sda` and `3` with your actual values):
   ```bash
   sudo parted /dev/sda set 3 esp off
   ```
   Any warning about `/etc/fstab` can be safely ignored.

> ⚠️ **Critical — Do NOT unmount the drive or `/cdrom`.**  
> The Subiquity installer backend requires these to remain actively mounted throughout the installation. Unmounting will crash the installer.

---

### Phase 4 — Manual Installation

> With the fake EFI flag removed, the installer no longer sees the temporary partition as a bootable EFI drive — GRUB installs cleanly to your real system drive.

1. Switch back to the installer and proceed through the setup screens until you reach **Installation Type**.

2. Select **Something else** (Manual Partitioning).

3. **Configure the root partition:**
   - Find the Unallocated space you created in Phase 1 → double-click it.
   - **Use as:** Ext4 journaling file system
   - ✅ **Format the partition**
   - **Mount point:** `/`
   - Click OK.

4. **Set the bootloader drive:**
   - At the **bottom-left** of the partitioning screen, locate the **Device for boot loader installation** dropdown.
   - Select the drive that contains your Windows installation (e.g. `/dev/nvme0n1` or `/dev/sda`).
   - The installer will automatically detect and use the existing Windows EFI partition on that drive — you do not need to configure the EFI partition manually.

   > ⚠️ **If you have two physical drives:**  
   > This dropdown will not auto-select. Identify which drive holds your Windows EFI partition (typically the primary drive) and select it explicitly. Choosing the wrong drive will break your Windows bootloader.

5. Click **Install Now** and complete the remaining setup.

---

### Phase 5 — Cleanup

1. Restart. You will be greeted by the **GRUB boot menu** — use it to choose between Ubuntu and Windows on every boot.

2. Boot into **Windows**.

3. Open **Disk Management** → right-click the `INSTALLER` partition → **Delete Volume**.

4. Right-click your `C:` drive → **Extend Volume** to absorb the freed 8 GB back into Windows.

🎉 **Done. Clean dual-boot — no USB, no external hardware, no leftover partitions.**

---

## Contributing

Tested on a different Ubuntu release or found a hardware-specific behaviour worth documenting? PRs and issues are welcome.

---

## License

MIT
