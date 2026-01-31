---
title: "Linux Mint 21.3 or 22 (LMDE) on Asus T100TA"
author: "Derk Lambers"
date: "2026-01-30"
tags: ["Linux", "Asus T100TA", "Mint22", "UEFI", "32-bit"]
---

# Linux Mint 21.3 or 22 (LMDE) on Asus T100TA  
## Definitive GRUB EFI-IA32 Installation & Repair Guide

This guide provides a **complete method** to install Linux Mint 22 or LMDE on the Asus T100TA, a device with a 64-bit CPU and 32-bit UEFI firmware. It includes all steps for USB prep, installation, GRUB, EFI entries and optional final tweaks.

---

## Why GRUB Fails on the Asus T100TA

The T100TA will only boot if:
- GRUB is installed as 32-bit EFI (i386-efi)
- A valid EFI boot entry exists
- The entry points to a real EFI file
- EFI variables (efivars) are mounted
- The OS is installed on internal MMC storage

---

## Requirements

- Asus T100TA
- Linux Mint 21.3 or 22 Live USB
- Installation completed
- Do NOT reboot after installation

---

## USB Preparation

1. Download **Linux Mint 22**.  
2. Use **Rufus (Windows)** or **Balena Etcher (Windows/Linux)** to write the ISO as **GPT + UEFI (without Secure Boot enabled in BIOS)**.  
3. Optionally copy `bootia32.efi` to the USB stick:

```bash
cp bootia32.efi /media/USB/EFI/BOOT/
```

> This ensures the T100TA can boot from the USB stick if the default loader fails.

---

## Step 0 — Installation

1. Boot the USB in **UEFI mode**. 
2. Connect to Wifi for getting the latest updates and third party drivers.
3. During installation:
```text
- Select "Erase disk and install Mint"
- Use entire eMMC or SSD
- Let the installer create EFI partition automatically
```
4. **Do not boot into Mint yet** — we will repair GRUB first.

---

## Step 1 — Identify partitions

```bash
lsblk -f
```

Typical layout:
- mmcblk1p1 = EFI (vfat)
- mmcblk1p3 = Linux root (ext4)

---

## Step 2 — Mount installed system

```bash
sudo mkdir -p /target
sudo mount /dev/mmcblk1p3 /target
```

---

## Step 3 — Bind essential directories

```bash
sudo bash -c '
for dir in dev dev/pts proc run sys sys/firmware/efi/efivars; do
    mount --bind /$dir /target/$dir
done'
```

---

## Step 4 — DNS fix

```bash
sudo cp /etc/resolv.conf /target/etc/resolv.conf
```

---

## Step 5 — Enter chroot

```bash
sudo chroot /target /bin/bash
```

---

## Step 6 — Mount EFI

```bash
mkdir -p /boot/efi
mount /dev/mmcblk1p1 /boot/efi
```

---

## Step 7 — Install GRUB packages

```bash
apt update
apt install grub-efi-ia32-bin efibootmgr
```

---

## Step 8 — Install GRUB

```bash
grub-install --target=i386-efi --efi-directory=/boot/efi --boot-directory=/boot --no-nvram
update-grub
```

---

## Step 9 — Verify EFI file

```bash
ls /boot/efi/EFI/debian/
```

Expect:
- grubia32.efi

---

## Step 10 — Create EFI boot entry

```bash
efibootmgr -c -d /dev/mmcblk1 -p 1 -l '\EFI\debian\grubia32.efi' -L 'Linux Mint'
```

---

## Step 11 — Exit & reboot

```bash
exit
cd /
sudo umount /target/sys/firmware/efi/efivars
sudo umount /target/dev/pts
sudo umount /target/dev
sudo umount /target/proc
sudo umount /target/run
sudo umount /target/sys
sudo umount /boot/efi
sudo umount /target
reboot
```

---

## Post-Install Fixes
Advanced performance and usability tweaks for the T100TA (Intel Atom, 2 GB RAM, eMMC).

### 1. Grub No Splash Screen

```bash
sudo nano /etc/default/grub
```
Change:
```text
GRUB_TIMEOUT=5
```
To:
```text
GRUB_TIMEOUT=0
```
Add:
```text
GRUB_TIMEOUT_STYLE=hidden
GRUB_HIDDEN_TIMEOUT=0
```
Then:
```bash
sudo update-grub2
sudo reboot
```

### 2. Reduce Swappiness (major performance gain)

```bash
echo "vm.swappiness=10" | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl -p /etc/sysctl.d/99-swappiness.conf
```

---

### 3. Enable ZRAM (compressed RAM swap)

```bash
sudo apt update
sudo apt install zram-tools
```

Verify:
```bash
swapon --show
```

### 4. Backlight Fix (brightness keys)

```bash
sudo nano /etc/default/grub
```

Edit:
```text
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_backlight=intel"
```

Apply:
```bash
sudo update-grub
sudo reboot
```

### 5. Enable TRIM for eMMC

```bash
systemctl status fstrim.timer
sudo systemctl enable fstrim.timer
sudo systemctl start fstrim.timer
```

### 6. Preload (Application Preloading)

```bash
sudo apt update
sudo apt install preload
sudo systemctl enable preload
sudo systemctl start preload
```

### 7. Limit systemd-journald Disk Usage

```bash
sudo nano /etc/systemd/journald.conf
```

Add or change:
```text
SystemMaxUse=50M
RuntimeMaxUse=20M
```

```bash
sudo systemctl restart systemd-journald
```

### 8. Reduce Disk Writes (noatime)

```bash
sudo nano /etc/fstab
```

Example entry:
```text
UUID=xxxx / ext4 defaults,noatime 0 1
```
Do not modify other fields unless you know what you are doing.

### 9. Boot Time Optimization

```bash
systemd-analyze blame
```

Often safe to disable if unused:
```bash
sudo systemctl disable ModemManager
sudo systemctl disable avahi-daemon
```

### 10. Low-RAM Kernel VM Tweaks

Create config:
```bash
sudo nano /etc/sysctl.d/99-lowram.conf
```

Add:
```text
vm.vfs_cache_pressure=50
vm.dirty_ratio=10
vm.dirty_background_ratio=5
```

Apply:
```bash
sudo sysctl -p /etc/sysctl.d/99-lowram.conf
```

### 11. Disable Unneeded Background Services

If not used:
```bash
sudo systemctl disable bluetooth
sudo systemctl disable cups
```