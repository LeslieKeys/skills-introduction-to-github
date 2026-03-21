# Touch Bar Driver Installation Guide for Ubuntu 24.04 on MacBook Pro 2017

This guide covers installing the `apple-ibridge` (Touch Bar) kernel module on a
**2017 MacBook Pro** running **Ubuntu 24.04** with Linux kernel **6.17**.

The driver source files in this repository include all patches from
[roadrunner2/macbook12-spi-driver PR #77](https://github.com/roadrunner2/macbook12-spi-driver/pull/77)
that fix compilation errors against kernel 6.17.

## What Was Fixed (kernel 6.17 compatibility)

The following changes were applied to make the driver compile cleanly on kernel 6.17:

| File | Fix |
|---|---|
| `applespi.c` | `asm/unaligned.h` → `linux/unaligned.h` |
| `applespi.c` | `delay_usecs` → `delay.value` + `delay.unit` (4 instances) |
| `applespi.c` | `no_llseek` → `noop_llseek` |
| `applespi.c` | EFI variable helpers rewritten to use `efivar_get_variable`/`efivar_set_variable` |
| `applespi.c` | `applespi_remove`/`appleacpi_remove`: `int` → `void` return type |
| `applespi.c` | Removed `.owner = THIS_MODULE` from `struct acpi_driver` |
| `applespi.c` | Added `MODULE_IMPORT_NS("EFIVAR")` |
| `apple-ibridge.c` | `report_fixup` return type: `__u8 *` → `const __u8 *` |
| `apple-ibridge.c` | `appleib_remove`: `int` → `void` return type |
| `apple-ibridge.c` | Removed `.owner` from `acpi_driver` |
| `apple-ib-tb.c` | `appletb_platform_remove`: `int` → `void` return type |
| `apple-ib-als.c` | `iio_device_alloc`/`iio_trigger_alloc` now require parent device as first argument |
| `apple-ib-als.c` | `iio_dev->id` → `iio_device_id(iio_dev)` |
| `apple-ib-als.c` | `appleals_platform_remove`: `int` → `void` return type |

---

## Prerequisites

Before you start, make sure you have the required build tools and kernel headers:

```bash
sudo apt update
sudo apt install git dkms build-essential linux-headers-$(uname -r)
```

---

## Step 1: Remove any previous broken installation

If you have an old, failed `apple-ibridge` DKMS installation, remove it first:

```bash
sudo dkms remove -m apple-ibridge -v 0.1 --all 2>/dev/null || true
sudo rm -rf /usr/src/apple-ibridge-0.1
sudo dkms remove -m applespi -v 0.1 --all 2>/dev/null || true
sudo rm -rf /usr/src/applespi-0.1
```

---

## Step 2: Install the patched driver

Copy this repository's contents to `/usr/src/applespi-0.1` and install via DKMS:

```bash
sudo cp -r /path/to/this/repo /usr/src/applespi-0.1
sudo dkms install -m applespi -v 0.1
```

Alternatively, clone directly from this repository (which contains the kernel 6.17
patched driver files):

```bash
sudo git clone https://github.com/LeslieKeys/skills-introduction-to-github.git /usr/src/applespi-0.1
sudo dkms install -m applespi -v 0.1
```

> **Note:** The driver source files in this repository are based on the
> [roadrunner2/macbook12-spi-driver](https://github.com/roadrunner2/macbook12-spi-driver)
> `touchbar-driver-hid-driver` branch with patches from
> [PR #77](https://github.com/roadrunner2/macbook12-spi-driver/pull/77)
> applied for kernel 6.17 compatibility.

A successful install produces output similar to:

```
Building module:
Cleaning build area...
make -j4 KERNELRELEASE=6.17.0-19-generic...
ECC module...done.
Signing module...done.
Depmod...done.
```

---

## Step 3: Add modules to initramfs

Ensure the SPI and Apple iBridge modules load automatically at boot:

```bash
sudo bash -c 'cat >> /etc/initramfs-tools/modules << EOF

# Apple SPI / Touch Bar modules
applespi
spi_pxa2xx_platform
intel_lpss_pci
apple-ibridge
apple-ib-tb
EOF'
sudo update-initramfs -u
```

---

## Step 4: Load the modules immediately (without rebooting)

```bash
sudo modprobe applespi
sudo modprobe apple-ibridge
sudo modprobe apple-ib-tb
sudo modprobe apple-ib-als
```

---

## Step 5: Verify the Touch Bar is working

### Check that modules loaded successfully

```bash
lsmod | grep -E 'applespi|apple_ibridge|apple_ib'
```

Expected output:

```
apple_ib_tb            49152  0
apple_ib_als           32768  0
apple_ibridge          65536  2 apple_ib_tb,apple_ib_als
applespi               98304  0
```

### Check kernel messages for errors

```bash
sudo dmesg | grep -i -E 'apple|iBridge|touchbar|applespi' | tail -20
```

A working installation shows lines like:

```
[   3.421] apple-ibridge: Apple iBridge device found
[   3.422] apple-ib-tb: Apple Touch Bar driver loaded
```

### Confirm the Touch Bar USB device is present

```bash
lsusb | grep -i apple
```

You should see at least one Apple USB entry corresponding to the iBridge chip.

### Check DKMS status

```bash
dkms status
```

Expected output:

```
applespi/0.1, 6.17.0-19-generic, x86_64: installed
```

---

## Troubleshooting

### DKMS build still fails

1. Check the build log for errors:
   ```bash
   cat /var/lib/dkms/applespi/0.1/build/make.log
   ```

2. Make sure you have the correct kernel headers installed:
   ```bash
   sudo apt install linux-headers-$(uname -r)
   ```

### Module loads but Touch Bar is blank

- Try toggling the function key mode:
  ```bash
  echo 2 | sudo tee /sys/bus/hid/drivers/apple-ib-tb/*/fnmode
  ```

- Check for HID device errors:
  ```bash
  sudo dmesg | grep -i hid | tail -20
  ```

### Modules not loading after reboot

Verify the `/etc/initramfs-tools/modules` file contains the module names and
run `sudo update-initramfs -u` again, then reboot.

---

## References

- [roadrunner2/macbook12-spi-driver](https://github.com/roadrunner2/macbook12-spi-driver) — upstream driver
- [PR #77: Fix compilation errors for Linux kernel 6.17](https://github.com/roadrunner2/macbook12-spi-driver/pull/77) — kernel 6.17 patches
- [Dunedan/mbp-2016-linux](https://github.com/Dunedan/mbp-2016-linux) — state of Linux on MacBook Pro 2016/2017
