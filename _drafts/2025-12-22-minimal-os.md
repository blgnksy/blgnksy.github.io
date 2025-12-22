---
layout: post
comments: true
title: Minimalist OS
lang: eng
header:
  teaser: "https://blgnksy.github.io/assets/img/linux/Tux.svg"
tags: [Linux, Busybox, Boot Sequence]
---

II have been using Linux as my daily driver—both professionally and personally—for quite a long time. My fascination with kernel internals has led me to build the kernel myself and boot it inside a virtual machine (I use QEMU). I’m planning another blog post about kernel compilation, but for now I’ll focus on how to create a bootable ISO image for Linux.

To test and debug my custom kernel I need a bootable ISO. The exact use‑cases will be defined later, but first let’s look at the general boot process:

1. **Firmware initialization (BIOS/UEFI)**  
   When the system powers on, the firmware runs POST, initializes hardware, and searches for a bootable device.  
   *Legacy BIOS* loads the first‑stage bootloader into RAM; modern systems with **UEFI** load an EFI application (e.g., `bootx64.efi`).

2. **Bootloader (GRUB, Windows Boot Manager, etc.)**  
   The bootloader switches the CPU to the appropriate execution mode, loads the Linux kernel image (`bzImage`) and the initial RAM filesystem (initramfs, typically `initrd.img`), and then transfers control to the kernel entry point.

3. **Kernel decompression and early initialization**  
   The kernel decompresses itself to its final location, jumps to the architecture‑specific entry point, and invokes `start_kernel()`. This routine sets up memory management, the scheduler, interrupt handling, core subsystems, and mounts the temporary root filesystem (the initramfs) in preparation for userspace.

4. **Early userspace (initramfs)**  
   The kernel runs the first userspace program, trying the following in order:  
   - `/init`  
   - `/sbin/init`  
   - `/etc/init`  
   - `/bin/init`  
   - `/bin/sh`  
   During this phase the initramfs populates `/dev` (typically via `udev`), loads any required kernel modules, and mounts the virtual filesystems `/proc` and `/sys`.

5. **Root filesystem handoff**  
   After the real root filesystem is ready, the kernel switches to it using `pivot_root` or `switch_root`, then re‑executes the init process from the new root. This marks the transition from early userspace to the main userspace environment.

6. **Init system startup**  
   PID 1 (usually `systemd`, but could be OpenRC, SysVinit, etc.) starts system services and configures console devices.

7. **Shell execution**  
   Once the init system finishes, a login shell is presented. The system is now fully booted, and any userspace program can be launched.

Now, we can start creating an ISO image. I am going to explain step by step with tested and working code/command examples. First, let's find a kernel image and initial RAM filesystem. Open a shell and list all files in `/boot` directory:
 
```bash
ls -la /boot
```
Output:

```bash
total 625628
dr-xr-xr-x. 6 root root      4096 Dec 21 09:31 .
dr-xr-xr-x. 1 root root       192 Dec 22 14:32 ..
-rw-r--r--. 1 root root    292973 Dec 13 01:00 config-6.17.12-300.fc43.x86_64
-rw-r--r--. 1 root root    292938 Oct  6 02:00 config-6.17.1-300.fc43.x86_64
drwx------. 4 root root      4096 Jan  1  1970 efi
drwx------. 3 root root      4096 Dec 22 15:36 grub2
-rw-------. 1 root root 270890527 Dec 20 15:51 initramfs-0-rescue-70a9f446bdd94bfb904e2762ff2f9046.img
-rw-------. 1 root root 146082930 Dec 21 09:31 initramfs-6.17.12-300.fc43.x86_64.img
-rw-------. 1 root root 146076670 Dec 21 09:30 initramfs-6.17.1-300.fc43.x86_64.img
drwxr-xr-x. 3 root root      4096 Dec 20 15:49 loader
drwx------. 2 root root     16384 Dec 20 15:46 lost+found
lrwxrwxrwx. 1 root root        47 Dec 20 16:17 symvers-6.17.12-300.fc43.x86_64.xz -> /lib/modules/6.17.12-300.fc43.x86_64/symvers.xz
lrwxrwxrwx. 1 root root        46 Dec 20 15:49 symvers-6.17.1-300.fc43.x86_64.xz -> /lib/modules/6.17.1-300.fc43.x86_64/symvers.xz
-rw-r--r--. 1 root root  11127277 Dec 13 01:00 System.map-6.17.12-300.fc43.x86_64
-rw-r--r--. 1 root root  12017392 Oct  6 02:00 System.map-6.17.1-300.fc43.x86_64
-rwxr-xr-x. 1 root root  17807720 Dec 20 15:50 vmlinuz-0-rescue-70a9f446bdd94bfb904e2762ff2f9046
-rwxr-xr-x. 1 root root  18184232 Dec 13 01:00 vmlinuz-6.17.12-300.fc43.x86_64
-rw-r--r--. 1 root root       162 Dec 13 01:00 .vmlinuz-6.17.12-300.fc43.x86_64.hmac
-rwxr-xr-x. 1 root root  17807720 Oct  6 02:00 vmlinuz-6.17.1-300.fc43.x86_64
-rw-r--r--. 1 root root       161 Oct  6 02:00 .vmlinuz-6.17.1-300.fc43.x86_64.hmac
```

The output can be (actually will be) diffent depending on your OS and its kernel version. I am using Fedora 43 in my personal computer. But, of course, good news, there is a pattern. The kernel image (vmlinuz‑<version>) is the file that we are interested in for now.

```bash
mkdir custom_iso && cd custom_iso
cp /boot/vmlinuz‑<version>
```

The plan is to have a system consists of a Linux kernel and a minimal BusyBox-based userspace in a custom initial filesystem. If you are not familiar with Busybox, you can check [its documentation](https://busybox.net/about.html). We are going to use it to have shell and run some of the utilities (Linux userspace programs) from its huge list. Let's now download its source code and statically compile it:

```bash
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
mkdir busybox-1.37.0
tar xvf busybox-1.37.0.tar.bz2 busybox-1.37.0/
make defconfig
sed 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/g' -i .config
LDFLAGS="--static" make -j$(nproc) busybox
```

Now, we can start preparing the root file system. Let's create a directory and get the root filesystem structure inside it:

```bash
mkdir -p initrd && cd initrd
mkdir -p bin dev proc sys root
```

Since busxybox is a single executable that a lot of Linux utilities are merged into, we are going to list all available utilities and create soft link for each in `/bin` directory:

```bash
cd bin
cp ../../busybox-1.37.0/busybox .
chmod +x busybox
for util in $(./busybox --list);do
  ln -s ./busybox ./"$util"
done
cd ..
```

We only need to craft our initialization script. We want out initialization script (our PID 1) to mount virtual filesystem, to provide device nodes, to set some kernel logging configuration up, and to run a shell, simple but yet powerful. But a more detailed list of responsibilities of a PID 1 are:
- Starts/monitors essential system processes,
- Handles the signals such as SIGTERM, SIGINT, SIGCHLD,
- Mounts virtual filesystems (/proc, /sys, /dev),
- Loads kernel modules, 
- Ensures /dev/console exists,
- Sets up stdin/stdout/stderr,
- Starts login or shell on consoles,
- Establishes base environment variables,
- Mounts the real root filesystem,
- Performs `switch_root` or `pivot_root`,
- Fees initramfs memory,
- Handles poweroff, reboot, halt requests,
- Invokes kernel reboot/poweroff,
- Handles Ctrl-Alt-Del behavior as well as `SIGPWR`, and `SIGWINCH`,

```bash
echo '#!/bin/sh' > init
echo 'mount -t sysfs sysfs /sys' >> init
echo 'mount -t proc proc /proc' >> init
echo 'mount -t devtmpfs udev /dev' >> init
echo 'sysctl -w kernel.printk="2 4 1 7"' >> init
echo 'clear' >> init
echo '/bin/sh' >> init
echo 'poweroff -f' >> init
```

Let's create our root filesystem archieve. We are going to `cpio` to create our archieve:

```bash
chmod -R 777 .
find . | cpio -o -H newc  > ../initrd.img
cd ..
```

We, now, have everything to create our bootable ISO image. We are going to use `grub` as bootloader:

```bash
# Copy the kernel and initial ramdisk
mkdir -p iso/boot/grub
cp bzImage iso/boot/
cp initrd.img iso/boot/

# Create GRUB config
cat > iso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

menuentry "Custom Linux with Busybox userspace" {
    linux /boot/bzImage
    initrd /boot/initrd.img
}
EOF

# Generate the ISO
grub-mkrescue -o custom_os.iso iso/
```

Let's test it using `qemu`:
```bash
qemu-system-x86_64 -m 4096 -smp 4  -kernel bzImage -initrd initrd.img
```

![](/assets/img/linux/busybox_boot.png)


The final step is to create a bootable ISO image. `grub`  is the bootloader and we are going to use `grub2-mkrescue` (check the executable for your OS) to create the ISO image:

```bash
# Create ISO directory structure
mkdir -p iso/boot/grub

# Copy the kernel and initial ramdisk
cp bzImage iso/boot/
cp initrd.img iso/boot/

# Create GRUB config
cat > iso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

menuentry "Custom Linux" {
    linux /boot/bzImage
    initrd /boot/initrd.img
}
EOF

# Generate the ISO
grub2-mkrescue -o custom_os.iso iso/
```

```bash
qemu-system-x86_64 -m 4096 -smp 6  --cdrom ./custom_os.iso -boot d
```

![](/assets/img/linux/busybox_boot_iso.png)