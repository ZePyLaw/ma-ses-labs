## 2. Configure U-Boot to boot Linux manually

### U-Boot version and interactive shell

The first objective was to confirm that the Raspberry Pi was really running the U-Boot version built during Lab 01 and that the UART connection allowed interaction with its command-line interface.

After interrupting the automatic boot process, the `version` command was used:

```text
version
```

The result was:

```text
U-Boot 2025.01
```

![U-Boot version](images/lab2-uboot-version.png)

This confirms that the expected U-Boot binary is executing on the target.

The interactive U-Boot shell is important because it allows the boot process to be decomposed manually. Instead of treating booting as one opaque operation, U-Boot lets us separately inspect storage, load files into RAM, modify kernel arguments, inspect the Device Tree and finally start the kernel.

This distinction is useful for debugging: if Linux fails to boot, it becomes possible to determine whether the problem occurred while reading storage, loading the kernel, configuring its arguments, passing the Device Tree, or mounting the root filesystem.

### Inspecting the SD card

U-Boot accesses SD/eMMC devices through its MMC subsystem.

The available devices were first listed with:

```text
mmc list
```

The Raspberry Pi exposed two MMC controllers, and device `0` was identified as the physical SD card:

```text
mmcnr@7e300000: 1
mmc@7e340000: 0 (SD)
```

The SD card was selected with:

```text
mmc dev 0
```

and inspected with:

```text
mmc info
```

The card reported a capacity of approximately **29.7 GiB**.

The partition table was then displayed with:

```text
part list mmc 0
```

![SD card partitions](images/lab2-sd-partitions.png)

The SD card contains an MBR/DOS partition table with two partitions:

| Partition | Start sector | Number of sectors | Type | Purpose |
| --- | ---: | ---: | --- | --- |
| `mmc 0:1` | 1 | 65536 | `0x0c` | FAT boot partition |
| `mmc 0:2` | 65537 | 245760 | `0x83` | Linux root filesystem |

The SD card itself is much larger than these partitions because the Buildroot image defines fixed-size filesystems. Flashing the image reproduces those filesystem sizes; it does not automatically expand them to occupy the entire physical SD card.

This is an important distinction:

- the **physical SD card** has a capacity of about 29.7 GiB;
- the **Buildroot image** only defines a roughly 32 MiB boot partition and a roughly 120 MiB root filesystem;
- the remaining card space is therefore unused.

### Filesystems and kernel location

The filesystem type of each partition was checked with:

```text
fstype mmc 0:1
fstype mmc 0:2
```

The result was:

```text
mmc 0:1 -> FAT
mmc 0:2 -> ext4
```

The contents were then inspected with:

```text
fatls mmc 0:1 /
ext4ls mmc 0:2 /
```

![SD filesystems and kernel](images/lab2-sd-filesystems-kernel.png)

The first partition contains files required during the boot process, including:

- Raspberry Pi firmware files such as `start4.elf`;
- Device Tree files such as `bcm2711-rpi-4-b.dtb`;
- `u-boot.bin`;
- the Linux kernel image `Image`.

The second partition contains the Linux root filesystem, with directories such as:

```text
/bin
/dev
/etc
/lib
/proc
/sbin
/sys
/usr
/var
```

This demonstrates an important Linux concept: **the kernel and the root filesystem are separate objects**.

The kernel is a single executable image loaded into RAM before Linux starts. The root filesystem contains the userspace environment that Linux mounts later as `/`.

The exact kernel file is therefore:

```text
Device:      mmc 0
Partition:   1
Filesystem:  FAT
Filename:    Image
```

Its size was obtained using:

```text
fatsize mmc 0:1 /Image
printenv filesize
```

which returned:

```text
filesize=1578a00
```

U-Boot represents many numerical values in hexadecimal. `0x1578a00` corresponds to **22,514,176 bytes**, approximately **21.5 MiB**.

### Inspecting the Device Tree

The Raspberry Pi firmware loads the Device Tree before U-Boot starts. U-Boot exposes the address of this Device Tree Blob through the environment variable `fdt_addr`.

The address was inspected with:

```text
printenv fdt_addr
```

which returned:

```text
fdt_addr=2eff2300
```

The FDT commands were then pointed at this address:

```text
fdt addr ${fdt_addr}
```

and the `/chosen` node was displayed with:

```text
fdt print /chosen
```

![Device Tree chosen node](images/lab2-fdt-chosen.png)

The `/chosen` node is particularly interesting because it contains information intended for the operating system, including the kernel command line stored in the `bootargs` property.

Among the existing arguments were:

```text
root=/dev/mmcblk0p2
rootwait
console=tty1
console=ttyAMA0,115200
```

Only these arguments were required for the manual boot performed in this part of the lab.

### Understanding the kernel arguments

The U-Boot environment variable conventionally used to pass the kernel command line is `bootargs`.

It was configured with:

```text
setenv bootargs 'root=/dev/mmcblk0p2 rootwait console=tty1 console=ttyAMA0,115200'
```

The resulting command line contains four important pieces of information.

#### `root=/dev/mmcblk0p2`

This tells Linux which block device contains its root filesystem.

The name can be decomposed as:

```text
mmcblk0   -> first MMC/SD block device
p2        -> partition 2
```

Therefore Linux will mount the second SD-card partition as `/`.

#### `rootwait`

The kernel may reach the root-filesystem mounting stage before the SD controller has completely initialized. `rootwait` tells Linux to wait until the root block device becomes available instead of immediately failing.

#### `console=tty1`

This enables a Linux console on the first virtual terminal, normally associated with the local display.

#### `console=ttyAMA0,115200`

This enables the serial console on the Raspberry Pi UART at 115200 baud.

Having the serial console is particularly useful in embedded development because kernel messages and the login prompt remain accessible even without a monitor or graphical environment.

### Loading the Linux kernel into RAM

Finding `Image` on the SD card is not enough to execute it. U-Boot must first copy the kernel from persistent storage into DRAM.

The Raspberry Pi U-Boot configuration provides the variable:

```text
kernel_addr_r=0x00080000
```

This variable identifies a RAM address intended for loading the kernel.

The kernel was loaded with:

```text
fatload mmc 0:1 ${kernel_addr_r} Image
```

![Kernel boot arguments and load](images/lab2-bootargs-kernel-load.png)

The command can be read as:

```text
fatload
    mmc             storage interface
    0:1             MMC device 0, partition 1
    ${kernel_addr_r} destination address in RAM
    Image           source filename
```

U-Boot reported:

```text
22514176 bytes read
```

This matches the kernel size found earlier, confirming that the entire `Image` file was copied from the FAT partition into RAM.

An important point for the oral exam is that `kernel_addr_r` does **not** describe where the kernel is stored on the SD card.

It describes where U-Boot should place the kernel **in RAM** before execution.

### Starting the kernel with `booti`

The ARM64 Linux kernel was finally started with:

```text
booti ${kernel_addr_r} - ${fdt_addr}
```

![Manual Linux boot](images/lab2-manual-linux-boot.png)

The three arguments have distinct purposes:

```text
${kernel_addr_r}   -> address of the Linux kernel in RAM
-                  -> no initial RAM disk (initrd) is used
${fdt_addr}        -> address of the Device Tree Blob
```

`booti` is the U-Boot command used to boot a Linux kernel stored in the ARM64 `Image` format.

During execution, U-Boot reported that it moved the image from `0x80000` to another suitable location before starting it. This relocation is handled internally by U-Boot and does not change the meaning of `kernel_addr_r`: that variable still specifies the address where the image is initially loaded.

The kernel then started successfully and eventually reached the configured Linux login prompt.

### What this manual boot demonstrated

By manually reproducing the boot sequence, the relationship between the major components became clearer:

```text
SD FAT partition
      |
      | fatload
      v
Linux Image in DRAM
      |
      | booti
      |
      +---- bootargs
      |
      +---- Device Tree at fdt_addr
      |
      v
Linux kernel
      |
      | root=/dev/mmcblk0p2
      v
ext4 root filesystem on SD partition 2
      |
      v
Linux userspace
```

The key idea is that U-Boot does not itself provide the Linux userspace. Its role is to prepare the information and objects that the kernel needs, then transfer control to Linux.

For this manual SD boot, U-Boot provided:

- the kernel image in RAM;
- the kernel command line through `bootargs`;
- the Device Tree through `fdt_addr`.

Linux then took over, initialized the hardware and mounted the root filesystem itself.

### Oral-exam notes

`Image` is the Linux kernel file stored on the FAT partition. `kernel_addr_r` is a RAM address used to load that file before execution. These are two completely different concepts.

`bootargs` is a string passed by U-Boot to the Linux kernel. It controls kernel behaviour; it is not a set of commands executed by U-Boot.

The Device Tree describes the hardware to Linux. On this Raspberry Pi configuration, the firmware loads it before U-Boot and exposes its address through `fdt_addr`.

The command:

```text
booti ${kernel_addr_r} - ${fdt_addr}
```

therefore means: boot the ARM64 kernel loaded at `kernel_addr_r`, without an initrd, using the Device Tree located at `fdt_addr`.
