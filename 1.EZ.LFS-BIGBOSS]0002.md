# BIGBOSS (bizbox) Linux From Scratch Notes for x86_64 EFI System

---

👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻

*Note: This guide follows a specific approach and may deviate from traditional methods.*

*Disclaimer: Only tested with Casper ink. Deviations may not adhere to "THE WAY."*

Reference: [Ubuntu Forums - LFS Guide](https://ubuntuforums.org/showthread.php?t=688872)

---

### Useful Links:

- [Boot Multiple ISO from USB via GRUB2](https://www.pendrivelinux.com/boot-multiple-iso-from-usb-via-grub2-using-linux/)
- [Compiling the Linux Kernel and Creating a Bootable ISO](https://medium.com/@ThyCrow/compiling-the-linux-kernel-and-creating-a-bootable-iso-from-it-6afb8d23ba22)
- [Booting GRUB USB Stick](https://checkmk.com/linux-knowledge/booting-grub-usb-stick)

📍

```bash
losetup -d /dev/loop15
```

🌩️🌩️🌩️🌩️🌩️🌩️🌩️**HOLY GHOST = FLASH FORTH**🌩️🌩️🌩️🌩️🌩️

⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡⚡

Erase flash, then `mkfs.vfat = uuid`, `blkid (/etc/fstab)`? (use "label" instead of "uuid") [Link](https://linuxconfig.org/how-to-label-hard-drive-partition-under-linux)

```bash
mkfs.exfat /dev/sda
sudo e2label /dev/sda "ff"
sudo mount -t vfat /dev/disk/by-label/MY_BACKUP /mnt
mount -t vfat /dev/disk/by-uuid/FDC2-F7B4 /mnt
sudo grub-install --no-floppy --force --root-directory=/mnt /dev/sda
cp -v live-cd.iso /mnt
```

⚡⚡⚡⚡⚡⚡⚡⚡⚡

👁 😉

👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻👻

(((Research GRUB or use "kexec" as bootloader))) *I'm locking this down*

*My character's solid director as "atomic roo(T)ms" for Hololens 😉 theory after this*
*(So don't forget to add this to SCR]011)*

---

### Goal of the Guide:

The goal is to provide a simplified version of the Linux From Scratch project. Steps are organized in short sections to allow testing progress at each stage.

Key Difference: Instead of building all packages and tools of a regular Linux OS, start with a basic system based on the kernel and BusyBox—a binary providing all required commands.

---

**Reference Links:**

- [Linux from Scratch Stable](https://www.linuxfromscratch.org/lfs/downloads/stable/LFS-BOOK-11.0-NOCHUNKS.html)
- [Busybox-based Linux Distro from Scratch](https://re-ws.pl/2020/11/busybox-based-linux-distro-from-scratch/)

---

## Disk Configuration

For this guide, we use a disk image, but it can be adapted for real hardware.

### Create Raw Disk Image (30GiB example):

```bash
$ dd if=/dev/zero of=disk.img bs=1G count=30 status=progress
```

### Create Disk Partitions:

```bash
$ fdisk disk.img
```

1. Create a new GPT partition table
2. Add a new 512MiB EFI partition
3. Add a new partition for the root filesystem with the remaining space
4. Write the partition table

![fdisk.gif](https://gist.githubusercontent.com/daniruiz/5c1998a30e0da956463e20f3c7e1ef23/raw/fdisk.gif)

#### Format the Partitions:

```bash
$ losetup -P -f disk.img
$ losetup -a
$ lsblk /dev/loop0
$ mkfs.vfat -F32 /dev/loop0p1
$ mkfs.ext4 /dev/loop0p2
```

#### Check the New Partitions:

```bash
$ fdisk -l /dev/loop0
```

#### Mount the Root Partition:

```bash
$ sudo mkdir -v /mnt/big
$ sudo mount -v /dev/loop0p2 /mnt/big
$ sudo chown -v $(whoami) /mnt/big
```

---

## Building the Initial Root Filesystem

### Creating the Directory Layout:

```bash
$ cd /mnt/big
$ mkdir -pv dev etc proc run sys var usr/{bin,lib,src,sbin,lib64}
$ ln -sv usr/lib lib
$ ln -sv usr/lib lib64
$ ln -sv usr/lib usr/lib64
$ ln -sv usr/bin bin
$ ln -sv usr/bin sbin
$ ln -sv usr/bin usr/sbin
```

### Installing Shared Libraries (glibc):

```bash
$ cd /mnt/big/usr/src
$ wget -O- http://ftp.gnu.org/gnu/libc/glibc-2.35.tar.xz | xz -dc | tar -x
$ cd glibc-2.35
$ mkdir build
$ cd build
$ ../configure --prefix=/usr --enable-kernel=5.15
$ make
$ make DESTDIR=/mnt/big install
```

### Getting BusyBox:

```bash
$ cd /mnt/big/usr/src
$ wget -O- https://www.busybox.net/downloads/busybox-1.34.0.tar.bz2 | bzip2 -dc | tar -x
$ cd busybox-1.34.0
$ make defconfig
$ make -j
```

Copy the BusyBox binary to the system's `/bin`

 directory and create links for each command it provides:

```bash
$ cp -v busybox /mnt/big/bin/
$ cd /mnt/big/bin/
$ for t in $(./busybox --list); do ln -sv busybox $t; done
```

Configure BusyBox as the init program:

```bash
$ cd /mnt/big/
$ ln -sv bin/busybox /mnt/big/init
```

### Try It with chroot:

```bash
$ sudo chroot /mnt/big /bin/sh
```

---

## Getting the Kernel:

### Compile the Kernel:

```bash
$ cd /mnt/big/usr/src
$ wget -O- https://www.kernel.org/pub/linux/kernel/v5.x/linux-5.15.tar.xz | xz -dc | tar -x
$ cd linux-5.15
$ make -j x86_64_defconfig
```

Configure kernel parameters before compiling:

```bash
$ sudo PARTUUID=$(blkid /dev/loop0p2 -s PARTUUID -o value)
$ echo "CONFIG_CMDLINE_BOOL=y" >> .config
$ echo "CONFIG_CMDLINE=\"root=PARTUUID=$PARTUUID rw rootwait\"" >> .config
$ echo "CONFIG_CMDLINE_OVERRIDE=n" >> .config
$ make -j$(nproc)
```

### Try the Kernel with Qemu:

```bash
$ cd /mnt/big/usr/src/linux-5.14.8
$ qemu-system-x86_64 -hda /dev/loop0 -kernel arch/x86/boot/bzImage
```

---

## Making the Disk Image Bootable

### Configure the EFI Boot:

Mount the EFI partition:

```bash
$ mkdir -pv /mnt/big/boot/efi/
$ sudo mount -v /dev/loop0p1 /mnt/big/boot/efi/
```

Copy the compiled bzImage kernel to the EFI partition:

```bash
$ sudo mkdir -pv /mnt/big/boot/efi/EFI/Boot/
$ sudo cp -v /mnt/big/usr/src/linux-5.15/arch/x86/boot/bzImage /mnt/big/boot/efi/EFI/linux/bootx64.efi
```

### Try the Bootable Disk Image:

To boot Qemu in EFI mode:

```bash
$ qemu-system-x86_64 -hda /dev/loop0 -bios OVMF.fd
```

You can also use the path to the raw disk image instead of the loop device.

![boot-test.gif](https://gist.githubusercontent.com/daniruiz/5c1998a30e0da956463e20f3c7e1ef23/raw/boot-test.gif)

---

# TODO:

- 7.2. Change files ownership
- 7.6. Creating Essential Files and Symlinks
- Mount proc, run and sys
- Proper configuration of glibc
- Users
- Sign efi binary
- Network
- Compiler
- Cool stuff 🤪️
