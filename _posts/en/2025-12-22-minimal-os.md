---
layout: post
comments: true
title: Minimalist OS
lang: eng
header:
  teaser: "https://blgnksy.github.io/assets/img/linux/Tux.svg"
tags: [Linux, Busybox, Boot Sequence]
---

I have been using Linux as my daily driver—both professionally and personally—for many years. My fascination with kernel internals has led me to build the kernel myself and boot it inside a virtual machine (I use QEMU). I’m planning another blog post about kernel compilation, but for now I’ll focus on how to create a bootable ISO image for Linux.

The example use‑cases of the need to create an ISO image will be defined later, but first let’s review the general boot process:

1. **Firmware initialization (BIOS/UEFI)**  
   When the system powers on, the firmware runs POST, initializes hardware, and looks for a bootable device.  
   *Legacy BIOS* loads the first‑stage bootloader into RAM; modern systems with **UEFI** load an EFI application (e.g., `bootx64.efi`).

2. **Bootloader (GRUB, Windows Boot Manager, etc.)**  
   The bootloader switches the CPU to the appropriate execution mode, loads the Linux kernel image (`bzImage`) and the initial RAM filesystem (initramfs, usually `initrd.img`), then transfers control to the kernel entry point.

3. **Kernel decompression and early initialization**  
   The kernel decompresses itself to its final location, jumps to the architecture‑specific entry point, and calls `start_kernel()`. This routine sets up memory management, the scheduler, interrupt handling, core subsystems, and mounts the temporary root filesystem (the initramfs) in preparation for userspace.

4. **Early userspace (initramfs)**  
   The kernel runs the first userspace program, trying the following in order:  

   - `/init`  
   - `/sbin/init`  
   - `/etc/init`  
   - `/bin/init`  
   - `/bin/sh`  

   During this phase the initramfs populates `/dev` (normally via `udev`), loads any required kernel modules, and mounts the virtual filesystems `/proc` and `/sys`.

5. **Root‑filesystem handoff**  
   Once the real root filesystem is ready, the kernel switches to it using `pivot_root` or `switch_root`, then re‑executes the init process from the new root. This marks the transition from early userspace to the main userspace environment.

6. **Init system startup**  
   PID 1 (usually `systemd`, but it could be OpenRC, SysVinit, etc.) starts system services and configures console devices.

7. **Shell execution**  
   After the init system finishes, a login shell is presented. The system is now fully booted, and any userspace program can be launched.

---
As I mentioned before, there would be some other use cases where a user needs to create an ISO image. 

## Use cases
1. Creating your own distro,
1. Creating system recovery and rescue environments,
1. Implementing/testing/debugging new kernel features
1. Educational and research needs
---

## Preparing the ISO

First, locate a kernel image and its initramfs. In a shell, list the contents of `/boot`:

```bash
ls -la /boot
Typical output (Fedora 43 example):

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
-rwxr-xr-x. 1 root root  18184232 Dec 13 01:00 vmlinuz-6.17.12-300.fc43.x86_64
-rwxr-xr-x. 1 root root  17807720 Oct  6 02:00 vmlinuz-6.17.1-300.fc43.x86_64
```

The file you’ll need now is the kernel image (vmlinuz‑<version>). Create a working directory and copy the kernel:
```bash
mkdir custom_iso && cd custom_iso
cp /boot/vmlinuz-6.17.12-300.fc43.x86_64 bzImage   # rename as you like
```

## Adding a Minimal BusyBox Userspace
If you’re not familiar with BusyBox, see its documentation: https://busybox.net/about.html. We’ll use it as a tiny, static userspace.

```bash
# Download and extract BusyBox
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
tar xf busybox-1.37.0.tar.bz2
cd busybox-1.37.0

# Default config → enable static linking
make defconfig
sed -i 's/# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config

# Build a statically linked binary
LDFLAGS="--static" make -j$(nproc) busybox
cd ..
Building the Initramfs
Create the minimal root filesystem hierarchy:

mkdir -p initrd/{bin,dev,proc,sys,root}
Populate /bin with BusyBox and symlinks for each provided utility:

cd initrd/bin
cp ../../busybox-1.37.0/busybox .
chmod +x busybox

# Create a symlink for every BusyBox applet
for util in $(./busybox --list); do
    ln -s ./busybox "$util"
done
cd ..
```
We only need to craft our initialization script. A detailed list of responsibilities of a PID 1 are:
- Starts/monitors essential system processes,
- Handles the signals such as `SIGTERM`, `SIGINT`, `SIGCHLD`,
- Mounts virtual filesystems (`/proc`, `/sys`, `/dev`),
- Loads kernel modules,
- Ensures `/dev/console` exists,
- Sets up `stdin`/`stdout`/`stderr`,
- Starts login or shell on consoles,
- Establishes base environment variables,
- Mounts the real root filesystem,
- Performs `switch_root` or `pivot_root`,
- Fees initramfs memory,
- Handles poweroff, reboot, halt requests, `Ctrl-Alt-Del` behavior as well as `SIGPWR`, and `SIGWINCH`,
- Invokes kernel reboot/poweroff.

We want out initialization script (our PID 1) to mount virtual filesystem, to provide device nodes, to set some kernel logging configuration up, and to run a shell.  

Write a very small init script (PID 1) that mounts the virtual filesystems and drops to a shell:
```bash
cat > init <<'EOF'
#!/bin/sh
mount -t sysfs sysfs /sys
mount -t proc proc /proc
mount -t devtmpfs udev /dev
sysctl -w kernel.printk="2 4 1 7"
clear
exec /bin/sh
EOF
chmod +x init
```
Package the initramfs using cpio (newc format is required by the Linux kernel):
```bash
chmod -R 777 .
find . | cpio -o -H newc > ../initrd.img
cd ..
```

## Testing the kernel and initial ram disk with qemu

```bash
qemu-system-x86_64 -m 4096 -smp 6  -kernel bzImage -initrd initrd.img 
```

You should be able to see the command prompt:
![](/assets/img/linux/busybox_boot.png)

Actually this is enough to test/debug the kernel. You don't need an ISO. 

## Creating the Bootable ISO with GRUB
```bash
# Directory layout expected by grub-mkrescue
mkdir -p iso/boot/grub

# Copy kernel and initramfs
cp bzImage iso/boot/
cp initrd.img iso/boot/

# GRUB configuration
cat > iso/boot/grub/grub.cfg <<'EOF'
set timeout=5
set default=0

menuentry "Custom Linux with BusyBox userspace" {
    linux /boot/bzImage
    initrd /boot/initrd.img
}
EOF

# Build the ISO
grub2-mkrescue -o custom_os.iso iso/ # On some distributions the command is grub-mkrescue; use whichever is available.
```
## Testing the ISO with QEMU
```bash
qemu-system-x86_64 -m 4096 -smp 4 -cdrom custom_os.iso -boot d
```

You should see a simple prompt provided by BusyBox:

![](/assets/img/linux/busybox_boot_iso.png)
