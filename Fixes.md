# AMD RC RAID Driver for Linux Kernel 6.14.x - Build & Installation Guide

## Overview

This guide documents the **fixes and procedures** to successfully compile and install the AMD RC RAID driver (version 9.3.2) on **Linux Kernel 6.14.x** and **Ubuntu 24.04/25.04**.

### What's New in This Guide

This README addresses critical compilation issues encountered with kernel 6.14+ and provides:
- **BTF Compilation Fix** - Solution for `Unsupported DW_TAG_reference_type` errors
- **Updated Makefile** - Modified to disable BTF generation for binary blob modules
- **Complete build and installation workflow** for kernel 6.14.0-29-generic and newer

---

## Prerequisites

### Required Packages

Install these packages before building:

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    linux-headers-$(uname -r) \
    dwarves \
    git \
    mokutil \
    openssl
```

### System Requirements

- **Kernel Version:** 6.14.0-29-generic or newer
- **Ubuntu Version:** 24.04 LTS or 25.04
- **Architecture:** x86_64
- **Disk Space:** ~500MB for build artifacts

---

## Issues Fixed

### 1. BTF (BPF Type Format) Compilation Error

**Problem:**
```
Unsupported DW_TAG_reference_type(0x10): type: 0x2a23f
Encountered error while encoding BTF.
make[4]: *** [scripts/Makefile.modfinal:57: rcraid.ko] Error 1
```

**Root Cause:**
Kernel 6.14+ enables `CONFIG_DEBUG_INFO_BTF_MODULES` by default. The BTF encoder (`pahole`) cannot process the proprietary binary blob (`rcblob.x86_64.o`) included in the rcraid driver, causing build failures.

**Solution:**
Modified `driver_sdk/src/Makefile` line 100 to disable BTF generation:

```makefile
# Original:
$(MAKE) -C $(KDIR) M=$(shell pwd) modules

# Fixed:
$(MAKE) -C $(KDIR) M=$(shell pwd) CONFIG_DEBUG_INFO_BTF_MODULES= modules
```

This change prevents the kernel build system from attempting BTF encoding on the module.

---

## Build Instructions

### Step 1: Clone the Repository

```bash
cd ~/Downloads
git clone https://github.com/Bemly/rcraid-driver-9.3.2-5.x-6.14.git
cd rcraid-driver-9.3.2-5.x-6.14/driver_sdk/src
```

**Note:** The kernel 6.14 compatibility patch is already applied in this repository.

### Step 2: Clean Previous Builds (if any)

```bash
make clean
```

### Step 3: Build the Driver

```bash
sudo make
```

**Expected Output:**
```
------------------------------------------------------------
- building for kernel 6.14.0-29-generic
------------------------------------------------------------
make -C /lib/modules/6.14.0-29-generic/build M=/path/to/src CONFIG_DEBUG_INFO_BTF_MODULES= modules
  CC [M]  rc_init.o
  CC [M]  rc_msg.o
  CC [M]  rc_mem_ops.o
  CC [M]  rc_event.o
  CC [M]  rc_config.o
  CC [M]  vers.o
  LD [M]  rcraid.o
  MODPOST Module.symvers
  CC [M]  rcraid.mod.o
  LD [M]  rcraid.ko

Signing module using local certificate
Chosen sha512 for signing
Signing /path/to/rcraid.ko Success
```

### Step 4: Verify Build

```bash
ls -lh rcraid.ko
modinfo rcraid.ko
```

**Expected:**
- File size: ~12MB
- Kernel version: `6.14.0-29-generic`
- Signature: `sig_id: PKCS#7`, `sig_hashalgo: sha256`

---

## Module Signing (Secure Boot)

### Automatic Signing

The build process **automatically** creates and signs the module with a local certificate stored in `/var/lib/rccert/certs/`.

### First-Time Setup (if keys don't exist)

If you've never built this driver before, the `mk_certs` script will:
1. Generate a 4096-bit RSA signing key
2. Create a self-signed X.509 certificate
3. Import the certificate into MOK (Machine Owner Key) database
4. Sign the `rcraid.ko` module

**You'll be prompted to set a password** - remember this! You'll need it during the next reboot to enroll the key.

### Verify Key Enrollment

Check if your signing key is already enrolled:

```bash
mokutil --test-key /var/lib/rccert/certs/module_signing_key.der
```

**If enrolled:** `module_signing_key.der is already enrolled`
**If not enrolled:** `module_signing_key.der is not enrolled`

### Manual Key Enrollment (if needed)

If the key is not enrolled:

```bash
sudo mokutil --import /var/lib/rccert/certs/module_signing_key.der
```

Enter a password when prompted. On next reboot:
1. MOK Manager will appear (blue screen)
2. Select **Enroll MOK**
3. Select **Continue**
4. Enter the password you just set
5. Select **Reboot**

---

## Installation

### For Live USB / Pre-Installation

If installing Ubuntu on a RAID array:

```bash
# 1. Load the driver in Live environment
sudo insmod rcraid.ko

# 2. Verify RAID disks are visible
lsblk

# 3. Proceed with Ubuntu installation

# 4. Post-install (before reboot)
sudo cp rcraid.ko /target/lib/modules/$(uname -r)/kernel/drivers/scsi/
sudo chroot /target
depmod -a
mkinitramfs -o /boot/initrd.img-$(uname -r) $(uname -r)
exit

# 5. Reboot
```

### For Existing Ubuntu Installation

If Ubuntu is already installed and you're updating the driver:

```bash
# 1. Copy module to kernel directory
sudo cp rcraid.ko /lib/modules/$(uname -r)/kernel/drivers/scsi/

# 2. Update module dependencies
sudo depmod -a

# 3. Backup existing initramfs (optional but recommended)
sudo cp /boot/initrd.img-$(uname -r) /boot/initrd.img-$(uname -r).backup

# 4. Rebuild initramfs
sudo mkinitramfs -o /boot/initrd.img-$(uname -r) $(uname -r)

# 5. Load the module (optional - to test before reboot)
sudo modprobe rcraid

# 6. Reboot to use RAID on boot
sudo reboot
```

### Verify Installation

After reboot:

```bash
# Check if module is loaded
lsmod | grep rcraid

# Check module info
modinfo rcraid

# Check RAID devices
lsblk
ls /dev/rc*
```

---

## Updating After Kernel Upgrade

If you upgrade your kernel (e.g., from 6.14.0-29 to 6.14.0-30):

```bash
cd ~/Downloads/rcraid-driver-9.3.2-5.x-6.14/driver_sdk/src

# Set new kernel version
export NEW_KVER=$(uname -r)

# Clean and rebuild
sudo make clean
sudo make KVERS=$NEW_KVER

# Install
sudo cp rcraid.ko /lib/modules/$NEW_KVER/kernel/drivers/scsi/
sudo depmod -a $NEW_KVER
sudo mkinitramfs -o /boot/initrd.img-$NEW_KVER $NEW_KVER

# Reboot
sudo reboot
```

---

## Troubleshooting

### Issue: BTF Error Still Occurs

**Symptom:**
```
Encountered error while encoding BTF.
make[4]: *** [scripts/Makefile.modfinal:57: rcraid.ko] Error 1
```

**Solution:**
Verify the Makefile was correctly modified:

```bash
grep "CONFIG_DEBUG_INFO_BTF_MODULES=" driver_sdk/src/Makefile
```

Should show:
```
$(MAKE) -C $(KDIR) M=$(shell pwd) CONFIG_DEBUG_INFO_BTF_MODULES= modules
```

If not, manually edit line 100 in `driver_sdk/src/Makefile`.

---

### Issue: Module Fails to Load - "Required key not available"

**Symptom:**
```
insmod: ERROR: could not insert module rcraid.ko: Required key not available
```

**Solution:**
Your signing key is not enrolled. See [Module Signing](#module-signing-secure-boot) section above.

---

### Issue: RAID Disks Not Visible

**Symptom:**
RAID arrays don't appear in `lsblk` or `/dev/`

**Solution:**
```bash
# Check if module loaded
lsmod | grep rcraid

# Check kernel logs for errors
dmesg | grep -i rcraid

# Try loading manually
sudo modprobe rcraid

# Check PCI devices
lspci | grep -i raid
```

---

### Issue: Frame Size Warning

**Symptom:**
```
rc_mem_ops.c:935:1: warning: the frame size of 1080 bytes is larger than 1024 bytes
```

**Solution:**
This is a **warning**, not an error. The module will still build successfully. This can be safely ignored.

---

## Technical Details

### Modified Files

1. **`driver_sdk/src/Makefile`** (Line 100)
   - Added `CONFIG_DEBUG_INFO_BTF_MODULES=` to build command
   - Prevents BTF encoding on proprietary binary blob

2. **`driver_sdk/src/rc_init.c`** (Already patched)
   - Added kernel 6.14 compatibility for `rc_slave_cfg()` function signature
   - Includes `<linux/vmalloc.h>` for kernel 6.10+

3. **`driver_sdk/mk_certs`** (No changes needed)
   - Certificate generation and module signing script

### Build Artifacts

After successful build:
- `rcraid.ko` - Main kernel module (~12MB)
- `Module.symvers` - Symbol version information
- `*.o` - Object files (rc_init.o, rc_msg.o, etc.)

### Signing Certificate Location

- **Private Key:** `/var/lib/rccert/certs/module_signing_key.priv` (permissions: 0600)
- **Public Certificate:** `/var/lib/rccert/certs/module_signing_key.der` (permissions: 0644)
- **OpenSSL Config:** `/var/lib/rccert/certs/x509.genkey`

---

## Testing

### Test Module Loading (Before Installation)

```bash
# Try loading without installing
sudo insmod ./rcraid.ko

# Check if loaded
lsmod | grep rcraid

# Check kernel messages
dmesg | tail -20

# Unload
sudo rmmod rcraid
```

### Verify Signature

```bash
modinfo rcraid.ko | grep sig_
```

Expected output:
```
sig_id:         PKCS#7
sig_key:        3E:DD:BF:F1:83:18:C2:11:A5:16:80:BE:BF:A4:9A:72:EB:A4:7D:BE
sig_hashalgo:   sha256
```

---

## Additional Resources

- **Original Driver Source:** [thopiekar/rcraid-dkms#44](https://github.com/thopiekar/rcraid-dkms/pull/44#issuecomment-1013929676)
- **Kernel 6.14 Patch:** [Bemly/rcraid-patch-932](https://github.com/Bemly/rcraid-patch-932)
- **Ubuntu Secure Boot Guide:** [Ubuntu SecureBoot](https://ubuntu.com/blog/how-to-sign-things-for-secure-boot)

---

## Known Limitations

1. **Binary Blob Dependency:** The driver includes a proprietary binary blob (`rcblob.x86_64.o`) which limits:
   - BTF debugging information
   - Source-level debugging
   - Distribution in kernel mainline

2. **Kernel Version Sensitivity:** May require updates for major kernel version changes (6.15+)

3. **Architecture:** Currently only supports x86_64

---

## License

This driver is proprietary software:
- Copyright © 2020-2023 Advanced Micro Devices, Inc.
- Use subject to AMD's written software license agreement
- NOT GPL or open source licensed

---

## Support

For issues specific to:
- **Driver compilation:** Check this README and open an issue in the repository
- **AMD RAID hardware:** Contact AMD Support
- **Ubuntu installation:** Check Ubuntu documentation

---

## Changelog

### 2025-10-04 - Kernel 6.14 BTF Fix
- **Fixed:** BTF compilation error on kernel 6.14+
- **Modified:** `Makefile` line 100 to disable `CONFIG_DEBUG_INFO_BTF_MODULES`
- **Tested:** Ubuntu 24.04 with kernel 6.14.0-29-generic
- **Status:** Build successful, module loads correctly

---

## Quick Reference Commands

```bash
# Build
cd rcraid-driver-9.3.2-5.x-6.14/driver_sdk/src
sudo make clean && sudo make

# Install
sudo cp rcraid.ko /lib/modules/$(uname -r)/kernel/drivers/scsi/
sudo depmod -a
sudo mkinitramfs -o /boot/initrd.img-$(uname -r) $(uname -r)

# Load
sudo modprobe rcraid

# Verify
lsmod | grep rcraid
dmesg | grep -i rcraid
lsblk
```

---

**Last Updated:** October 4, 2025
**Tested On:** Ubuntu 24.04 LTS, Kernel 6.14.0-29-generic
