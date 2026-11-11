---
title: "U-Boot Hands-On: Booting Linux on a Raspberry Pi 4"
description: "Three labs on the U-Boot console: boot Linux manually, automate it with bootcmd, and inspect the Device Tree. Plus the boot commands (bootm, bootz, booti, ...) and which kernel image type each one expects."
date: 2026-11-11 09:10:00 +0100
categories: [Embedded Linux]
tags: [u-boot, raspberry-pi, bootcmd, device-tree, bootm, booti, bootz, embedded-linux]
mermaid: true
---

This article is the practical companion of [Embedded Linux Boot Process](/posts/embedded-linux-boot-process/). We use **U-Boot 2024.01** on a **Raspberry Pi 4 Model B**, booting from the SD card, and go through three labs:

1. Boot Linux **manually**, one command at a time.
2. **Automate** the boot with `bootcmd`.
3. Inspect the **Device Tree** from the U-Boot console.

## Lab 1: Booting manually

The goal is to do by hand what U-Boot normally does on its own:

```mermaid
flowchart LR
    A["mmc dev"] --> B["fatls"] --> C["fatload kernel"] --> D["fatload DTB"] --> E["bootm"] --> F(["Linux"])
```

### Check the MMC devices

```text
U-Boot> mmc list
mmcnr@7e300000: 1
mmc@7e340000: 0 (SD)
U-Boot> mmc dev 0
switch to partitions #0, OK
mmc0 is current device
U-Boot> mmc info
Device: mmc@7e340000
Manufacturer ID: 11
OEM: 1111
Name: Bus Speed: 50000000
Mode: SD High Speed (50MHz)
Rd Block Len: 512
SD version 2.0
High Capacity: Yes
Capacity: 29.3 GiB
Bus Width: 4-bit
Erase Group Size: 512 Bytes
```

| Command | Role |
|---------|------|
| `mmc list` | Lists the MMC/SD controllers detected by U-Boot. Here, device 0 is the SD card. |
| `mmc dev 0` | Selects device 0 as the current MMC device, so later commands act on the SD card. |
| `mmc info` | Shows the card details: speed mode, block length, capacity, bus width. |

### View the files

```text
U-Boot> fatls mmc 0:1
 30696 bcm2708-rpi-b-plus.dtb
 ...
 56156 bcm2711-rpi-4-b.dtb
 ...
   263 boot.scr
 52476 bootcode.bin
   104 cmdline.txt
  2533 config.txt
 ...
8540888 uImage
 16384 uboot.env

40 file(s), 1 dir(s)
```

*(The listing is shortened: the real output also contains the other board DTBs, the `fixup*.dat` and `start*.elf` firmware files, and the `overlays/` directory.)*

`fatls mmc 0:1` lists the files of a FAT filesystem. The syntax is `mmc <device>:<partition>`, here device 0, partition 1. You just told U-Boot to *look at the FAT partition of the SD card and display its content*.

### Load the kernel

First, where should the kernel go in RAM? U-Boot keeps that address in an environment variable:

```text
U-Boot> printenv kernel_addr_r
kernel_addr_r=0x00080000
```

`kernel_addr_r` is the RAM address where U-Boot will load the Linux kernel. When we write `${kernel_addr_r}` in a command, U-Boot replaces it with its value. Later, the boot command uses the same address to find and start the kernel.

```text
U-Boot> fatload mmc 0:1 ${kernel_addr_r} uImage
8540888 bytes read in 367 ms (22.2 MiB/s)
```

`fatload mmc 0:1 <addr> <file>` reads a file from the FAT partition and copies it into RAM at the given address. The kernel is now in RAM.

### Load the Device Tree

```text
U-Boot> printenv fdt_addr_r
fdt_addr_r=0x02600000
U-Boot> fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb
56156 bytes read in 13 ms (4.1 MiB/s)
```

The DTB describing the Raspberry Pi 4 hardware is loaded at `fdt_addr_r`, away from the kernel. Now, in RAM:

```text
RAM
├── uImage              (at kernel_addr_r = 0x00080000)
└── bcm2711-rpi-4-b.dtb (at fdt_addr_r    = 0x02600000)
```

### Start Linux

```text
U-Boot> bootm ${kernel_addr_r} - ${fdt_addr_r}
Booting kernel from Legacy Image at 00080000 ...
   Image Name:   Linux-6.6.63-v7l
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    8540824 Bytes = 8.1 MiB
   Load Address: 00008000
   Entry Point:  00008000
   Verifying Checksum ... OK
   Flattened Device Tree blob at 02600000
   Booting using the fdt blob at 0x2600000
   Working FDT set to 2600000
   Loading Kernel Image
   Using Device Tree in place at 02600000, end 02610b5b
   Working FDT set to 2600000

Starting kernel ...
```

- `bootm <kernel> - <fdt>` boots a legacy **uImage**. The first argument is the kernel address, the `-` means *no initramfs*, and the last argument is the DTB address.
- `Verifying Checksum ... OK`: `bootm` checks the integrity of the image before using it.
- `Starting kernel ...`: U-Boot hands control to Linux, passing it the Device Tree.

We did it manually: `mmc dev` → `fatls` → `fatload` uImage → `fatload` DTB → `bootm` → Linux.

## Which boot command for which kernel image?

In Lab 1 we used `bootm` because our kernel is a **uImage**. But `bootm` is only one of several boot commands, and the right one depends on the **format of the kernel image** you load.

### The image types

| Image | What it is | Typical platforms |
|-------|-----------|-------------------|
| **uImage** | A kernel (often a `zImage`) wrapped by `mkimage` with a 64-byte *legacy* header: name, type, compression, **load address**, **entry point**, CRC checksums. | ARM 32-bit, historical default of U-Boot |
| **FIT image** (`.itb`) | *Flattened Image Tree*: one file holding the kernel, one or several DTBs, an optional initramfs, with hashes and signatures, and several *configurations*. | Any, used for verified boot |
| **zImage** | The raw ARM 32-bit kernel, compressed and **self-extracting**. No U-Boot header. | ARM 32-bit |
| **Image** | The raw, uncompressed kernel. No U-Boot header. | ARM 64-bit (AArch64), RISC-V |
| **bzImage** | The compressed x86 kernel. | x86, x86_64 |

### The commands

| Command | Boots | Syntax |
|---------|-------|--------|
| `bootm` | uImage (legacy), FIT image | `bootm [addr [initrd[:size]] [fdt]]` |
| `bootz` | zImage (ARM 32-bit) | `bootz [addr [initrd[:size]] [fdt]]` |
| `booti` | Image (ARM 64-bit, RISC-V) | `booti [addr [initrd[:size]] [fdt]]` |
| `zboot` | bzImage (x86) | `zboot [addr] [size] [initrd] [initrd_size]` |
| `bootefi` | An EFI application (GRUB, systemd-boot, an EFI-stub kernel) | `bootefi <addr> [fdt]` |
| `bootelf` | An ELF binary (bare-metal firmware, RTOS) | `bootelf [addr]` |
| `boot` | Runs the `bootcmd` variable (alias of `bootd`) | `boot` |

Three practical rules follow:

- **The command must match the image.** A raw `zImage` fed to `bootm` is rejected (`Bad Magic Number`), because `bootm` looks for a uImage or FIT header. The same file goes to `bootz`.
- **The three main commands share the same shape:** `<cmd> <kernel_addr> <initrd_addr or -> <fdt_addr>`. Only the first word changes, so converting a boot script from one image type to another is usually a one-word edit.
- **`boot` does not load a kernel by itself.** It simply runs whatever `bootcmd` contains, and that usually ends with one of the commands above (see Lab 2).

### Examples

```text
# uImage or FIT image (ARM 32-bit)
=> bootm ${kernel_addr_r} - ${fdt_addr_r}

# FIT image, selecting a configuration by name
=> bootm ${kernel_addr_r}#conf-rpi4

# zImage (ARM 32-bit)
=> bootz ${kernel_addr_r} - ${fdt_addr_r}

# Image (ARM 64-bit)
=> booti ${kernel_addr_r} - ${fdt_addr_r}

# With an initramfs: address:size (here the size of the last loaded file)
=> booti ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r}
```

### What changes with the image type

- **Load address.** A uImage carries its own *Load Address* and *Entry Point* in its header (`00008000` above), and `bootm` places the kernel there. A `zImage` or an ARM64 `Image` has no header, so it runs from where you loaded it: the address in `kernel_addr_r` matters, and for an ARM64 `Image` it must respect the alignment the kernel expects. That is why boards define `kernel_addr_r` in their default environment.
- **Integrity.** `bootm` verifies the uImage CRC (`Verifying Checksum ... OK`), and with a FIT image it can also verify hashes and signatures. `bootz` and `booti` have no header to check.
- **Initramfs.** With `bootm` on a legacy uImage, the initramfs must itself be a U-Boot ramdisk image. With `bootz` and `booti`, you pass a raw initramfs as `address:size`.
- **Device Tree.** All three accept the DTB address as the last argument. With a FIT image, the DTB can come from the FIT itself.

> Not sure which command to use? Type `help bootm`, `help bootz` or `help booti` at the U-Boot prompt. Only the commands compiled into your U-Boot build are available.
{: .prompt-tip }

## Lab 2: Automating with bootcmd

We no longer want to type all these commands. We want:

```text
=> boot
```

and U-Boot does everything automatically.

### Create a variable for the boot

```text
U-Boot> setenv myboot 'fatload mmc 0:1 ${kernel_addr_r} uImage; fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb; bootm ${kernel_addr_r} - ${fdt_addr_r}'
```

- `setenv <name> '<value>'` creates or modifies an environment variable. Several commands can be chained with `;`.
- The **single quotes** keep `${kernel_addr_r}` and `${fdt_addr_r}` unexpanded until the variable is run.

> Type the `setenv` command on **one line** in the U-Boot console. It is wrapped over several lines in this article only for readability.
{: .prompt-info }

### Verify

```text
U-Boot> printenv myboot
myboot=fatload mmc 0:1 ${kernel_addr_r} uImage; fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb; bootm ${kernel_addr_r} - ${fdt_addr_r}
```

### Test without saving

`run myboot` executes the content of a variable as a sequence of commands. The change is only in RAM and is lost at reset.

```text
U-Boot> run myboot
8540888 bytes read in 367 ms (22.2 MiB/s)
56156 bytes read in 13 ms (4.1 MiB/s)
Booting kernel from Legacy Image at 00080000 ...
   ...
Starting kernel ...
```

### Modify bootcmd

`bootcmd` is the variable U-Boot runs automatically at startup, after the autoboot delay. We make it run our `myboot` variable, then `boot` runs `bootcmd` exactly as the autoboot does:

```text
U-Boot> setenv bootcmd 'run myboot'
U-Boot> printenv bootcmd
bootcmd=run myboot
U-Boot> boot
```

### Save

- `saveenv` writes the environment to persistent storage (here `uboot.env` on the FAT partition), so it survives a reboot.
- `reset` restarts the board. U-Boot reloads the saved environment and boots through `bootcmd` unless a key is pressed.

```text
U-Boot> setenv bootcmd 'run myboot'
U-Boot> saveenv
Saving Environment to FAT... OK
U-Boot> reset
resetting ...

U-Boot 2024.01 (Jan 08 2024 - 15:37:48 +0000)
DRAM:  948 MiB (effective 1.9 GiB)
RPI 4 Model B (0xb03115)
...
Loading Environment from FAT... OK
...
Hit any key to stop autoboot:  0
8540888 bytes read in 366 ms (22.3 MiB/s)
56156 bytes read in 14 ms (3.8 MiB/s)
## Booting kernel from Legacy Image at 00080000 ...
   ...
Starting kernel ...
```

The board now boots Linux by itself, with no typing: `Loading Environment from FAT... OK`, then autoboot, then `bootcmd`, then `myboot`.

## Lab 3: Device Tree in U-Boot

### Load the DTB

```text
U-Boot> fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb
56156 bytes read in 14 ms (3.8 MiB/s)
U-Boot> fdt addr ${fdt_addr_r}
Working FDT set to 2600000
```

- `fatload mmc 0:1 ${fdt_addr_r} ...` loads the DTB from the SD card into RAM.
- `fdt addr ${fdt_addr_r}` tells U-Boot where the Device Tree is located in RAM and sets it as the **working Device Tree** for the other `fdt` commands. `fdt` is the U-Boot command to manipulate the Device Tree, `addr` sets its address, and `${fdt_addr_r}` holds that address.

### Display the Device Tree

```text
U-Boot> fdt print /
/ {
    compatible = "raspberrypi,4-model-b";
    model = "Raspberry Pi 4 Model B";
    ...
};
```

`fdt print /` prints the whole Device Tree, starting from the root node `/`. You can also print a single node by its path:

```text
U-Boot> fdt print /memory
memory@0 {
    device_type = "memory";
    reg = <0x00000000 0x00000000 0x00000000>;
};
```

## Summary

| Goal | Commands |
|------|----------|
| Look at the storage | `mmc list`, `mmc dev`, `mmc info`, `fatls` |
| Load files into RAM | `fatload mmc 0:1 <addr> <file>` |
| Know where to load | `printenv kernel_addr_r`, `printenv fdt_addr_r` |
| Start the kernel | `bootm` (uImage, FIT), `bootz` (zImage), `booti` (Image) |
| Automate | `setenv myboot '...'`, `setenv bootcmd 'run myboot'`, `boot` |
| Persist | `saveenv`, `reset` |
| Inspect the Device Tree | `fdt addr`, `fdt print` |
