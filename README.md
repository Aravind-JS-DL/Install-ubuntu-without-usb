# Install-ubuntu-dualboot-without-usb

> **Install Ubuntu alongside Windows 10/11 — no USB drive, no CD, no external hardware.**
> A modern, verified guide for UEFI/GPT systems.

## Table of Contents

- [Disclaimer](#️-disclaimer)
- [Why This Guide Exists](#why-does-this-guide-exist)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Phase 1 — Partition Your Drive in Windows](#phase-1--partition-your-drive-in-windows)
- [Phase 2 — Boot the Installer](#phase-2--boot-the-installer)
- [Phase 2B — If You Can't Boot (EFI Flag Workaround)](#phase-2b--if-you-cant-boot-efi-flag-workaround)
- [Phase 3 — Strip the EFI Flag (if applied)](#phase-3--strip-the-efi-flag-if-applied)
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

---

## How It Works

1. An **8 GB FAT32 partition** is created and the Ubuntu ISO contents are copied onto it — giving your system a bootable installer living entirely on your internal drive.
2. Most modern UEFI systems will detect and boot this partition directly from the BIOS boot menu.
3. If your motherboard refuses to boot it, a one-time **EFI GUID flag** can be assigned to the partition to force recognition — then immediately stripped once the live environment is running, keeping the GRUB installation clean.

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

## Phase 1 — Partition Your Drive in Windows

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

## Phase 2 — Boot the Installer

1. Restart your PC and repeatedly press your BIOS boot menu key (`F12`, `F8`, `F2`, or `Del` — varies by manufacturer).

2. From the boot menu, look for your **internal drive** listed as a secondary **"UEFI OS"** entry and select it.

3. If it appears and boots successfully into the Ubuntu installer, **skip Phase 2B entirely** and proceed to Phase 4.

> **Can't see the drive in the boot menu, or it fails to boot?** Your motherboard may require the official EFI GUID to recognise the partition as bootable. Follow Phase 2B to apply it.

---

## Phase 2B — If You Can't Boot (EFI Flag Workaround)

> This step is only needed if your motherboard refuses to boot the installer partition. It temporarily assigns an EFI system partition GUID to trick the firmware into recognising it.

1. Open **Command Prompt as Administrator**:
   Start → type `cmd` → right-click → **Run as administrator**

2. Run:
   ```cmd
   diskpart
   list volume
   ```

3. Locate your `INSTALLER` volume and note its **Volume #** (e.g. `4`).

4. Run the following, replacing `X` with your volume number:
   ```cmd
   select volume X
   set id=c12a7328-f81f-11d2-ba4b-00a0c93ec93b
   exit
   ```

> **What is that GUID?**
> `c12a7328-f81f-11d2-ba4b-00a0c93ec93b` is the universal EFI System Partition identifier defined by the UEFI specification. Assigning it signals your motherboard to treat the partition as bootable, regardless of what's actually on it.

5. Restart and enter your BIOS boot menu again. Your internal drive should now appear as a bootable **"UEFI OS"** entry — select it.

6. Select **Try or Install Ubuntu** and continue to Phase 3.

---

## Phase 3 — Strip the EFI Flag (if applied)

> **Skip this phase if you did not apply the EFI flag in Phase 2B.**

Once the live environment has loaded, you must remove the fake EFI flag before the installer runs — otherwise GRUB will try to install to the temporary partition instead of your real system drive.

1. Open Terminal: `Ctrl` + `Alt` + `T`

2. Identify your `INSTALLER` partition:
   ```bash
   lsblk
   ```
   Look for the ~8 GB partition. Note the **drive name** (e.g. `sda`, `nvme0n1`) and **partition number** (e.g. `3`).

3. Strip the EFI flag (replace `sda` and `3` with your actual values):
   ```bash
   sudo parted /dev/sda set 3 esp off
   ```
   Any warning about `/etc/fstab` can be safely ignored.

> ⚠️ **Do NOT unmount the drive or `/cdrom`.**
> Subiquity requires these to remain mounted throughout installation. Unmounting will crash the installer.

---

## Phase 4 — Manual Installation

1. Switch to the installer and proceed through setup until you reach **Installation Type**.

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

## Phase 5 — Cleanup

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
