# Knowledge Base: Expanding Disk Size on a Linux VM (Ubuntu) After Virtual Disk Resize

## Overview

This KB explains how to expand a Linux root partition and filesystem after the virtual disk has already been increased from the hypervisor layer (Hyper-V, VMware, Nutanix, cloud platform, etc.).

This procedure is commonly used when:

* VM disk size has been increased from the virtualization platform
* OS still shows the old partition size
* Additional free space is unallocated at the end of the disk

---

## Prerequisites

Before starting:

* Confirm the virtual disk has already been expanded from the hypervisor/storage layer
* Ensure you have sudo/root access to the server
* Recommended: take a VM snapshot / backup before making partition changes

---

# Step 1 — Verify Current Disk Layout

Check disk and partition sizes:

```bash
lsblk
```

Example:

```bash
NAME   SIZE TYPE MOUNTPOINT
sda    128G disk
├─sda1   1G part /boot/efi
└─sda2  59G part /
```

This indicates:

* Disk = 128G
* Root partition still smaller than disk
* Remaining space available to extend

---

# Step 2 — Check Filesystem Type

Identify whether the filesystem is `ext4` or `xfs`:

```bash
df -Th /
```

Example:

```bash
Filesystem     Type  Size  Used Avail Mounted on
/dev/sda2      ext4   58G   ... /
```

---

# Step 3 — Install Required Utility

Install `growpart` if not already available:

```bash
sudo apt update
sudo apt install -y cloud-guest-utils
```

---

# Step 4 — Expand the Partition

Grow the target partition to consume remaining free disk space.

Example for partition 2:

```bash
sudo growpart /dev/sda 2
```

Expected output:

```bash
CHANGED: partition=2 ... old: size=... new: size=...
```

---

# Step 5 — Expand the Filesystem

## For ext4 Filesystems

```bash
sudo resize2fs /dev/sda2
```

---

## For XFS Filesystems

```bash
sudo xfs_growfs /
```

---

# Step 6 — Validate Expansion

Verify updated disk size:

```bash
lsblk
```

Verify mounted filesystem:

```bash
df -h /
```

Expected:

```bash
/dev/sda2   ~full expanded size
```

---

# Troubleshooting

## growpart returns `NOCHANGE`

If `growpart` reports `NOCHANGE`, force Linux to rescan the disk:

```bash
echo 1 | sudo tee /sys/class/block/sda/device/rescan
```

Then retry:

```bash
sudo growpart /dev/sda 2
```

---

## Verify kernel detected capacity change

Check kernel logs:

```bash
sudo dmesg | tail -20
```

Look for:

```bash
sda: detected capacity change ...
EXT4-fs: resized filesystem ...
```

---

# Notes

* In most cases this can be completed online without reboot
* Filesystem resize for ext4 is typically non-disruptive while mounted
* If the server hosts critical applications, perform during a maintenance window as per change policy
* Always validate application health after disk expansion

---

# Example Quick Commands (ext4)

```bash
sudo apt install -y cloud-guest-utils
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
lsblk
df -h /
```

---

# Outcome

After completion:

* Virtual disk size recognized by Linux
* Partition expanded
* Filesystem resized
* Additional storage available to OS without rebuilding or remounting

