
---
layout: post
title: Installing Hafnium on PINE64-LTS
---

Hafnium is a type-1 hypervisor that provides memory isolation between the virtual machines that run on top of it. It is installed directly on top of the underlying hardware. In this tutorial, I will be showing how to setup hafnium on the PINE64-LTS. 



## Firmware

Before we can do anything we need to setup our secondary program loader, EL3 runtime firmware, and bootloader.



#### ARM TF-A

First, we need to download our ARM TF-A. This is responsible for the EL3 runtime secure monitor in the Pine A64-LTS. It switches between the normal and secure worlds.

```bash
git clone https://review.trustedfirmware.org/TF-A/trusted-firmware-a
cd trusted-firmware-a
git checkout v2.2
make PLAT=sun50i_a64 DEBUG=1 bl31
export BL31=$(pwd)/build/sun50i_a64/debug/bl31.bin
```

#### U-Boot

Now that we have our trusted firmware downloaded, we can download U-Boot our normal world bootloader. 

```bash
git clone https://gitlab.denx.de/u-boot/u-boot.git
cd u-boot
git checkout v2020.04-rc3
make pine64_plus_defconfig
make 
export PATH=$PATH:$(pwd)/tools/
```

ARM TF-A and U-Booot are packaged together in u-boot-sunxi-with-spl.bin. We will copy this before the first partition on our sdcard at an 8kb offset later.

## Hafnium RAM Disk

Before we can Hafnium, we need to create a Hafnium RAM disk. Create a folder on your disk called hrdisk. Inside hrdisk create a file called manifest.dts and add this to it.

#### Mainfest

```
/dts-v1/;

/ {
	hypervisor {
		compatible = "hafnium,hafnium";
		vm1 {
			debug_name = "Linux VM";
			kernel_filename = "vmlinuz";
			ramdisk_filename = "initrd.img";
		};
	};
};
```

This manifest describes the VM to hafnium. Right now we are creating one VM with the linux kernel.

![Screen Shot 2021-07-01 at 8.21.38 PM](https://fdoku.me/images/h1.png)

The next step is to compile the dts file.

```bash
dtc -O dtb -o manifest.dtb -I dts manifest.dts 
```

#### Linux

Now we need to build our kernel. In the hrdisk directory run the following commands.

```bash
git clone https://github.com/torvalds/linux.git
cd linux
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j24
```

#### Hafnium Kernel Module

Alright, let's build the hafnium kernel module. Go back to the hrdisk folder and run

```bash
sudo apt install make libssl-dev flex bison python3 python3-serial
```

```bash
git clone --recurse-submodules https://hafnium.googlesource.com/hafnium && (cd hafnium && f=`git rev-parse --git-dir`/hooks/commit-msg ; curl -Lo $f https://gerrit-review.googlesource.com/tools/hooks/commit-msg ; chmod +x $f)
```

In order to compile the Hafnium kernel module for newer kernels you need to delete lines 765 and 766 of main.c.

```bash
cd hafnium/driver/linux/
```

![Screen Shot 2021-07-03 at 3.47.53 PM](https://fdoku.me/images/h2.png)

```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- KERNEL_PATH=../../../linux/ make
```

The hrdisk directory should now look like this.

![Screen Shot 2021-07-03 at 3.42.52 PM](https://fdoku.me/images/h3.png)

#### Busybox

We need a shell for our linux kernel. Let's create a file system for our Linux RAM disk with the Busybox shell. Head back to the hrdisk and run

```bash
git clone git://busybox.net/busybox.git
cd busybox
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make defconfig
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make menuconfig
```

Before continuing, make sure `Settings > Build static binary (no shared libs)` is selected. 

![Screen Shot 2021-07-03 at 3.53.55 PM](https://fdoku.me/images/h4.png)

Now in the same directory run these commands.

```bash
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make -j24
ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- make install
cd _install
mkdir proc
mkdir sys
mkdir -p etc/init.d
cat <<EOF > etc/init.d/rcS
#!bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
EOF
chmod u+x etc/init.d/rcS
grep -v tty ../examples/inittab > ./etc/inittab
```

In the _install directory run 

```bash
cp ../../hafnium/driver/linux/hafnium.ko .
```

to copy the hafnium kernel module to our Linux RAM disk.

Run the following commands to complete the Linux RAM disk.

```bash
find . | cpio -o -H newc | gzip > ../initrd.img
cd ..
```

#### Building

Now that we have everything we need to. Let's create the Hafnium RAM disk. Go back to hrdisk and run.

```bash
mkdir disk
cd disk
mv ../busybox/initrd.img .
mv ../manifest.dtb .
mv ../linux/arch/arm64/boot/Image vmlinuz
find . | cpio -o > ../initrd.img; cd -
mkimage -A arm64 -T ramdisk -C none -n uInitrd -d initrd.img uInitrd

```

We now have a hafnium RAM disk!

## Adding Support for PINE64-LTS

Now that we have hafnium let's add support for our PINE64-LTS board. From the hrdisk folder run

```bash
cd hafnium/project/reference
```

Now open `BUILD.gn` and add these lines to it.

```bash
"//src:hafnium(:pine64_clang)"
```

![Screen Shot 2021-07-03 at 4.05.38 PM](/Users/friedrichdoku/Desktop/Screen Shot 2021-07-03 at 4.05.38 PM.png)

Add these lines to the very bottom of the file.

```bash
aarch64_toolchains("pine64") {
  cpu = "cortex-a53"
  origin_address = "0x4a000000"
  boot_flow = "//src/boot_flow:linux"
  console = "//project/reference/pine:uart"
  iommu = "//src/iommu:absent"
  gic_version = 2
  heap_pages = 60
  max_cpus = 4
  max_vms = 16
  toolchain_args = {
    uart_base_address = "0x01C28000"
  }                 
} 
```

In the same folder create two new folders pine and pine64. 

```bash
mkdir pine
mkdir pine64
cd pine
touch args.gni BUILD.gn uart.c
```

In args.gni add these lines.

```bash
declare_args() {
  uart_base_address = 0
}
```

In BUILD.gn add these lines.

```bash
import("args.gni")

source_set("uart") {
  sources = [
    "uart.c",
  ]

  assert(uart_base_address != 0,
         "\"uart_base_address\" must be defined for ${target_name}.")

  defines = [
    "UART_BASE=${uart_base_address}",
  ]
}
```

In uart.c add these lines.

```c
#include "hf/io.h"
#include "hf/mm.h"
#include "hf/mpool.h"
#include "hf/plat/console.h"

/* clang-format off */

//#define UART_BASE (0x01C28000)

#define UART_RBR IO32_C(UART_BASE + 0x00)
#define UART_THR IO32_C(UART_BASE + 0x00)
#define UART_DLL IO32_C(UART_BASE + 0x00)
#define UART_IER IO32_C(UART_BASE + 0x04)
#define UART_DLM IO32_C(UART_BASE + 0x04)
#define UART_FCR IO32_C(UART_BASE + 0x08)
#define UART_LCR IO32_C(UART_BASE + 0x0C)
#define UART_MCR IO32_C(UART_BASE + 0x10)
#define UART_LSR IO32_C(UART_BASE + 0x14)
#define UART_MSR IO32_C(UART_BASE + 0x18)
#define UART_SCR IO32_C(UART_BASE + 0x1C)
/* clang-format on */

void plat_console_init(void)
{
}

void plat_console_mm_init(struct mm_stage1_locked stage1_locked,
			  struct mpool *ppool)
{
	mm_identity_map(stage1_locked, pa_init(UART_BASE),
                       pa_add(pa_init(UART_BASE), PAGE_SIZE),
                        MM_MODE_R | MM_MODE_W | MM_MODE_D, ppool);
}

void plat_console_putchar(char c)
{
	/* Print a carriage-return as well. */
	if (c == '\n') {
		plat_console_putchar('\r');
	}

	/* Wait until the transmitter is no longer busy. */
	while ((io_read32(UART_LSR) & 0x20) == 0) continue;

	/* Write data to transmitter FIFO. */
	memory_ordering_barrier();
	io_write32(UART_THR, c);
	memory_ordering_barrier();

}

char plat_console_getchar(void)
{
	/* Wait for the transmitter to be ready to deliver a byte. */
	while ((io_read32(UART_LSR) & 0x01) == 0) continue;

	/* Read data from transmitter FIFO. */
	return (char)(io_read32(UART_RBR));
} 
```

Now go to the pine64 folder.

```bash
cd ..
cd pine64
touch BUILD.gn
```

In the BUILD.gn file add these lines.

```bash
source_set("pine64") {
}
```

Now go back to the hafnium directory and run make.

```bash
make
```

## Booting Hafnium

Let's create some paritions on our sd card. The first partition will be our boot partition the second one can be used to create a root file system, but that will not be covererd in this tutorial.

```bash
sudo dd if=/dev/zero of=/dev/sdX bs=1M count=1
sudo blockdev --rereadpt /dev/sdX

cat <<EOT | sudo sfdisk /dev/sdX
2M,2048M,c
,,L
EOT

sudo mkfs.ext4 /dev/sdX1
sudo mkfs.ext4 /dev/sdX2
```

Go back to your u-boot folder and run.

```bash
dd if=u-boot-sunxi-with-spl.bin of=/dev/sdx bs=1k seek=8
```

Now let's mount the boot parition and copy the required files. Go [here](https://pitt-my.sharepoint.com/personal/frd20_pitt_edu/_layouts/15/onedrive.aspx?id=%2Fpersonal%2Ffrd20%5Fpitt%5Fedu%2FDocuments%2FEmbedded%20Systems&originalPath=aHR0cHM6Ly9waXR0LW15LnNoYXJlcG9pbnQuY29tLzpmOi9nL3BlcnNvbmFsL2ZyZDIwX3BpdHRfZWR1L0VnS2lRX2paNGtKSnRVSzREcWQwXzljQjM1ZENranBibTNCTE8yNS1BeDN1TlE%5FcnRpbWU9ODBQOXplY18yVWc) and download  the file `hafnium_pine64-lts_boot.tar.gz`. Once you have download the file, decompress it and copy the files from the boot directory to /dev/sdX1.

```bash
sudo mount /dev/sdX1 /mnt
tar -xvf hafnium_pine64-lts_boot.tar.gz
cp -r ./boot /mnt
rm /mnt/hafnium.bin
rm /mnt/uInitrd

cp ~/hrdisk/hafnium/out/reference/pine64_clang/hafnium.bin /mnt
cp ~/hrdisk/uInitrd /mnt

```

You are all set to go! Eject your sdcard and boot the PINE64-LTS board.
