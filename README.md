# 🔐 Ubuntu LUKS2 + dm-crypt Full Disk Encryption Guide

## 📌 Overview

This guide explains how to install **Ubuntu Desktop with full-disk encryption** using:

* 🔐 **LUKS2** — Linux Unified Key Setup
* 🔑 **dm-crypt** — Linux kernel block-device encryption
* 💾 **ext4** — Linux filesystem
* 🧩 **UEFI + GPT** — modern boot configuration
* 🔒 **Secure Boot** — recommended when supported
* 🖥️ **Ubuntu Desktop**
* 🪟 **i3wm** — optional, after the Ubuntu installation

Ubuntu uses LUKS for block-level disk encryption. LUKS2 is the modern LUKS format and is supported by current Ubuntu `cryptsetup` tooling.

---

# ⚠️ Important Warning

This guide assumes a **clean installation**.

Choosing to erase and encrypt a disk will permanently delete the existing data on that disk.

Before installation:

* Back up important files.
* Verify your backups.
* Make sure you know which disk you are going to erase.
* Keep your encryption passphrase in a secure place.

**Never run disk-partitioning commands against a disk containing data you need.**

---

# 1. Recommended Disk Layout

A typical encrypted Ubuntu installation looks like this:

```text
/dev/nvme0n1
├── /dev/nvme0n1p1    EFI System Partition
└── /dev/nvme0n1p2    LUKS2 encrypted partition
    └── dm-crypt
         └── Ubuntu root filesystem
```

Conceptually:

```text
UEFI
 │
 ├── EFI System Partition
 │       FAT32
 │
 └── LUKS2
      │
      └── dm-crypt
           │
           └── ext4
                │
                └── Ubuntu
                     │
                     └── i3wm
```

The EFI System Partition is normally not encrypted because firmware and the boot chain need access to it.

---

# 2. Create Ubuntu Installation USB

Download the official Ubuntu Desktop ISO and create a bootable USB drive.

Use a trusted source:

https://ubuntu.com/download/desktop

You can create the USB using tools such as:

* Rufus
* balenaEtcher
* GNOME Disks
* `dd`

---

# 3. Boot in UEFI Mode

Boot the computer from the Ubuntu USB.

Enter the firmware boot menu and select the USB device.

Prefer:

```text
UEFI: Ubuntu USB
```

instead of legacy BIOS/CSM mode.

After starting the live environment, open a terminal and check:

```bash
ls /sys/firmware/efi/efivars
```

If the directory exists, Ubuntu was booted in UEFI mode.

---

# 4. Start the Ubuntu Installer

Start:

```text
Install Ubuntu
```

Select your:

* Language
* Keyboard layout
* Network
* Time zone

Continue until you reach:

```text
Disk setup / Storage
```

---

# 5. Enable Disk Encryption

For a clean installation, select the option to:

```text
Erase disk and install Ubuntu
```

Then enable the encryption option, typically shown as:

```text
Encrypt the disk
```

or an equivalent full-disk encryption option depending on the Ubuntu installer version.

Ubuntu's installer provides password-based FDE options, and newer Ubuntu releases may also provide TPM-backed FDE on supported hardware.

---

# 6. Create the Encryption Passphrase

Create a strong encryption passphrase.

Example structure:

```text
River-Glass-Planet-47-Moon
```

Do **not** use this exact example as your real password.

Prefer a long passphrase that is:

* Long
* Unique
* Not reused anywhere else
* Easy for you to recover from your password manager or secure backup

Avoid:

```text
12345678
password
ubuntu
qwerty
```

---

# 7. Confirm Disk Erasure

Before pressing:

```text
Install
```

carefully check the selected disk.

For example:

```text
/dev/nvme0n1
```

If the computer contains multiple disks, **double-check the target disk**.

The installer will create the required partitions and encrypted storage.

---

# 8. Complete the Ubuntu Installation

Continue through the installer.

Create your normal Linux user account.

For example:

```text
Username: domino
```

Do not use the root account for everyday work.

Finish the installation and reboot.

Remove the USB drive when requested.

---

# 9. First Boot

During startup, the encrypted system may request the disk encryption passphrase.

Enter the LUKS encryption password you created during installation.

The boot process is approximately:

```text
UEFI
 ↓
Bootloader
 ↓
Encrypted storage
 ↓
LUKS2
 ↓
dm-crypt
 ↓
Ubuntu
 ↓
Login screen
```

---

# 10. Verify the Encryption

After logging into Ubuntu, open Terminal.

Run:

```bash
lsblk -f
```

You should see an encrypted partition with:

```text
crypto_LUKS
```

and a mapped device underneath it.

Example:

```text
nvme0n1
├─nvme0n1p1
└─nvme0n1p2
  └─sda_crypt
```

The exact device names may differ.

---

# 11. Find the LUKS Device

Run:

```bash
lsblk -o NAME,FSTYPE,UUID,SIZE,MOUNTPOINTS
```

Look for:

```text
crypto_LUKS
```

For example:

```text
nvme0n1p2   crypto_LUKS
```

---

# 12. Verify LUKS Version

Install `cryptsetup` if necessary:

```bash
sudo apt update
sudo apt install cryptsetup
```

Then:

```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```

Replace:

```text
/dev/nvme0n1p2
```

with your actual LUKS device.

Look for:

```text
Version:        2
```

This confirms that the volume uses **LUKS2**.

Modern Ubuntu `cryptsetup` supports explicitly creating LUKS2 with `--type luks2`, and LUKS2 is automatically recognized when activated.

---

# 13. Verify dm-crypt

Check the device mapper:

```bash
ls /dev/mapper/
```

You should see an encrypted mapping.

You can also run:

```bash
lsblk
```

Typical structure:

```text
nvme0n1
├─nvme0n1p1
└─nvme0n1p2
    └─ubuntu--vg-ubuntu--lv
```

or another Ubuntu-specific mapper name.

The exact layout depends on the Ubuntu installer version and whether LVM is used.

---

# 14. Check Active Device-Mapper Devices

Run:

```bash
sudo dmsetup ls
```

You may see something similar to:

```text
ubuntu--vg-ubuntu--lv
```

This indicates that the encrypted block device is being mapped through the Linux device-mapper subsystem.

---

# 15. Check Cryptsetup Status

If you know the crypt device name:

```bash
sudo cryptsetup status cryptroot
```

If Ubuntu uses another name, first inspect:

```bash
ls /dev/mapper/
```

Then use the corresponding mapping name.

---

# 16. Check the Root Filesystem

Run:

```bash
findmnt /
```

Example:

```text
TARGET SOURCE
/      /dev/mapper/ubuntu--vg-ubuntu--lv
```

This shows which device provides the root filesystem.

---

# 17. Check the Filesystem

Run:

```bash
df -Th /
```

Example:

```text
Filesystem                         Type  Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  ext4  200G   20G  170G  11% /
```

The filesystem may be `ext4` or another filesystem depending on your installation choices.

---

# 18. Check Encryption Information

Run:

```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```

Important information includes:

```text
Version:        2
PBKDF:
Keyslots:
UUID:
```

LUKS2 provides multiple key slots, allowing additional passphrases to be added and individual keys to be revoked.

---

# 19. Add an Additional LUKS Password

You can add another passphrase:

```bash
sudo cryptsetup luksAddKey /dev/nvme0n1p2
```

You will first be asked for an existing valid passphrase.

Then enter the new passphrase.

Verify:

```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```

Check the:

```text
Keyslots:
```

section.

---

# 20. Remove a LUKS Password

If you need to remove a passphrase:

```bash
sudo cryptsetup luksRemoveKey /dev/nvme0n1p2
```

Be extremely careful.

Make sure at least one working key remains.

Do not remove your only valid key.

---

# 21. Back Up the LUKS Header

A LUKS header backup is extremely important.

Create one:

```bash
sudo cryptsetup luksHeaderBackup \
    /dev/nvme0n1p2 \
    --header-backup-file luks-header-backup.img
```

Move the backup somewhere safe.

For example:

```text
External USB
Offline storage
Encrypted backup
Secure backup server
```

Do **not** keep the only copy on the encrypted disk.

The LUKS header contains the metadata and key-slot information required to access the encrypted container. Damage to the header can make the encrypted data inaccessible without a valid backup.

---

# 22. Protect the LUKS Header Backup

The header backup should be treated as sensitive data.

Recommended:

```text
LUKS encrypted disk
        │
        ├── LUKS header backup
        │
        └── Stored separately
```

Keep at least two secure copies.

For example:

```text
Copy 1 → Offline USB
Copy 2 → Secure encrypted backup
```

---

# 23. Check Secure Boot

Check Secure Boot status:

```bash
mokutil --sb-state
```

If `mokutil` is missing:

```bash
sudo apt install mokutil
```

Then:

```bash
mokutil --sb-state
```

You may see:

```text
SecureBoot enabled
```

or:

```text
SecureBoot disabled
```

Secure Boot is recommended when supported, especially when using TPM-backed FDE features. Ubuntu's newer TPM-backed FDE approach ties disk-unlock secrets to trusted boot measurements on supported hardware.

---

# 24. Check TPM 2.0

Check for TPM:

```bash
ls /dev/tpm*
```

You may see:

```text
/dev/tpm0
/dev/tpmrm0
```

You can also install:

```bash
sudo apt install tpm2-tools
```

Then:

```bash
sudo tpm2_getcap properties-fixed
```

If your hardware and Ubuntu installation support TPM-backed FDE, this can provide an alternative to manually entering the disk password during every boot. Availability depends on the Ubuntu release, installer path, and hardware support.

---

# 25. Important Difference: LUKS2 vs dm-crypt

These terms are related but not identical.

```text
LUKS2
 │
 └── Encryption metadata + key management
       │
       └── dm-crypt
             │
             └── Linux kernel encryption layer
```

**LUKS2** provides:

* Encryption metadata
* Key slots
* UUID
* PBKDF configuration
* Key management

**dm-crypt** provides the kernel-level block-device encryption mechanism.

For normal Ubuntu installations, you generally want **LUKS2 + dm-crypt**, rather than using plain dm-crypt directly. The Ubuntu `cryptsetup` documentation specifically recommends LUKS when you do not have a reason to use plain dm-crypt.

---

# 26. Optional: Install i3wm

If you want to use i3wm instead of GNOME:

```bash
sudo apt update
sudo apt install i3 i3status dmenu
```

For a useful basic setup:

```bash
sudo apt install \
    i3 \
    i3status \
    dmenu \
    picom \
    feh \
    dunst \
    kitty \
    network-manager-gnome \
    pavucontrol
```

Enable NetworkManager:

```bash
sudo systemctl enable NetworkManager
```

Reboot:

```bash
sudo reboot
```

You can then select **i3** from the login session menu.

---

# 27. Recommended Security Configuration

For a security-focused Ubuntu installation:

```text
UEFI
 │
 ├── Secure Boot
 │
 ├── EFI System Partition
 │
 └── LUKS2
      │
      └── dm-crypt
           │
           └── Ubuntu
                │
                └── i3wm
```

Recommended:

* ✅ UEFI
* ✅ GPT
* ✅ Secure Boot
* ✅ LUKS2
* ✅ dm-crypt
* ✅ Strong unique passphrase
* ✅ LUKS header backup
* ✅ Additional recovery key
* ✅ Regular backups
* ✅ Automatic security updates
* ✅ Firewall
* ✅ Separate normal user account
* ✅ No unnecessary services

---

# 28. Enable the Firewall

Ubuntu normally uses UFW as a convenient firewall frontend.

Check status:

```bash
sudo ufw status
```

Enable it:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status verbose
```

For a desktop system, the default configuration is usually sufficient unless you intentionally expose services.

---

# 29. Keep Ubuntu Updated

Update package information:

```bash
sudo apt update
```

Upgrade installed packages:

```bash
sudo apt upgrade
```

Or:

```bash
sudo apt update && sudo apt full-upgrade
```

Reboot after kernel updates when necessary:

```bash
sudo reboot
```

---

# 30. Verify Everything

Run:

```bash
lsblk -f
```

Then:

```bash
sudo cryptsetup luksDump /dev/nvme0n1p2
```

Then:

```bash
findmnt /
```

Then:

```bash
df -Th /
```

And:

```bash
mokutil --sb-state
```

A properly configured system should show:

```text
LUKS
 ↓
LUKS2
 ↓
dm-crypt
 ↓
filesystem
 ↓
/
```

---

# 🔐 31. Recovery Strategy

Disk encryption is not a backup system.

LUKS protects data at rest, but it does not protect against:

* SSD failure
* Accidental deletion
* Malware
* Ransomware
* Filesystem corruption
* Lost passwords
* Damaged LUKS metadata

Use a **3-2-1 backup strategy**:

```text
3 copies of important data
2 different types of storage
1 copy stored off-site
```

Keep the LUKS header backup separately from the encrypted disk.

---

# ⚠️ 32. Do Not Lose Your Encryption Password

If your installation uses password-based LUKS unlocking, losing the encryption passphrase can mean losing access to the encrypted data.

Store the passphrase securely.

A password manager plus an offline emergency backup is preferable to writing it in an unsecured text file.

---

# 📋 Final Architecture

```text
┌───────────────────────────────┐
│            UEFI               │
├───────────────────────────────┤
│       Secure Boot             │
├───────────────────────────────┤
│     EFI System Partition      │
│          FAT32                │
├───────────────────────────────┤
│           LUKS2               │
│      Encrypted Storage        │
├───────────────────────────────┤
│          dm-crypt             │
├───────────────────────────────┤
│       Ubuntu / LVM            │
├───────────────────────────────┤
│          ext4 /               │
├───────────────────────────────┤
│           i3wm                │
└───────────────────────────────┘
```

---

# ✅ Security Checklist

* [x] UEFI
* [x] GPT
* [x] LUKS2
* [x] dm-crypt
* [x] Encrypted root filesystem
* [x] Strong encryption passphrase
* [x] LUKS header backup
* [x] Additional recovery key
* [x] Secure Boot
* [x] TPM 2.0 where supported
* [x] UFW firewall
* [x] Regular system updates
* [x] 3-2-1 backups
* [x] i3wm support

---

# 📚 Official Documentation

* Ubuntu Desktop:
  https://ubuntu.com/desktop

* Ubuntu Full Disk Encryption:
  https://ubuntu.com/core/features/full-disk-encryption

* Ubuntu Full Disk Encryption Documentation:
  https://documentation.ubuntu.com/core/explanation/full-disk-encryption/

* Ubuntu `cryptsetup` documentation:
  https://manpages.ubuntu.com/manpages/noble/man8/cryptsetup.8.html

* Ubuntu Community — Full Disk Encryption:
  https://help.ubuntu.com/community/FullDiskEncryptionHowto
