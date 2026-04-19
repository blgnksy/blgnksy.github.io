---
layout: post
comments: true
title: Debugging Linux Kernel with QEMU and GDB
lang: en
header:
  teaser: "https://blgnksy.github.io/assets/img/linux/Tux.svg"
tags: [Linux, Kernel Debugging, Kernel Compilation, QEMU]
---
This article covers the fundamentals of compiling and debugging the Linux kernel using QEMU and GDB. The goal is to build a debug-friendly kernel image, run it in a virtual machine, and attach GDB to inspect boot flow, crashes, and kernel state.

## Compiling Linux Kernel for debugging
1. Clone the Linux kernel source:

```bash
git clone --branch v7.0 --depth 1 https://github.com/torvalds/linux.git
```

2. Use your current running kernel config as a starting point. This preserves driver and subsystem selections from your host environment.

```bash
cd linux
cp /boot/config-6.17.12-300.fc43.x86_64 .config
```

3. Update the configuration and select debugging options:

```bash
make olddefconfig
make menuconfig
```

`make olddefconfig` loads `.config` and fills any missing symbols with default values. Then use `make menuconfig` to inspect and enable debug options under the `Kernel hacking` menu.

4. Useful debug configuration options

- `CONFIG_DEBUG_INFO=y`: Generate DWARF debug symbols for GDB.
- `CONFIG_DEBUG_INFO_DWARF5=y`: Use DWARF version 5 when supported by your toolchain.
- `CONFIG_DEBUG_KERNEL=y`: Enable kernel debug checks and diagnostics.
- `CONFIG_FRAME_POINTER=y`: Preserve frame pointers for reliable stack traces.
- `CONFIG_CC_OPTIMIZE_FOR_DEBUG=y`: Reduce compiler optimizations to make code easier to step through.
- `CONFIG_GDB_SCRIPTS=y`: Install helper scripts for kernel debugging in GDB.
- `CONFIG_DEBUG_FS=y`: Enable the debug filesystem for runtime diagnostics.
- `CONFIG_KASAN=y`: Enable Kernel Address Sanitizer for memory error detection.
- `CONFIG_LOCKDEP=y`: Enable lock dependency tracking for deadlock debugging.
- `CONFIG_KGDB=y`: Support the kernel's own remote debugger backend.
- `CONFIG_EARLY_PRINTK=y`: Enable early printk output before normal console initialization.

For subsystem-specific bugs, enable additional options such as `CONFIG_DEBUG_SLAB`, `CONFIG_SCHED_DEBUG`, or the relevant driver debug options.

5. Build the kernel and keep the uncompressed ELF for debugging.

```bash
make -j$(nproc)
```

The bootable image appears at `arch/x86/boot/bzImage`. The debug binary `vmlinux` is the uncompressed ELF file that GDB loads.

## Running the kernel in QEMU
1. Start QEMU with a GDB server and serial console:

```bash
qemu-system-x86_64   -kernel arch/x86/boot/bzImage   -m 6G   -smp 4   -s -S   -append "console=ttyS0 nokaslr earlyprintk=serial,ttyS0,115200 panic=-1"   -nographic
```

QEMU options explained:
- `-kernel arch/x86/boot/bzImage`: Boot this kernel image directly.
- `-m 6G`: Allocate 6 GiB of RAM.
- `-smp 4`: Use 4 virtual CPUs.
- `-s`: Start a GDB server on TCP port `1234`.
- `-S`: Pause execution at startup until GDB connects.
- `-append "..."`: Pass kernel command-line parameters.
- `console=ttyS0`: Send kernel messages to the serial port.
- `nokaslr`: Disable kernel address-space layout randomization.
- `earlyprintk=serial,ttyS0,115200`: Enable very early serial logging.
- `panic=-1`: Prevent the kernel from rebooting after a panic.
- `-nographic`: Disable the graphical display and use the terminal for serial I/O.

Alternative options:
- `-gdb tcp::1234`: Explicitly set the GDB server port.
- `-display none`: Disable any video output.
- `-serial mon:stdio`: Combine the QEMU monitor and serial console on the same terminal.

2. In another terminal, start GDB with the uncompressed kernel image:

```bash
gdb vmlinux
```

3. Connect to QEMU and set breakpoints:

```gdb
(gdb) target remote :1234
(gdb) symbol-file vmlinux
(gdb) break start_kernel
(gdb) break panic
(gdb) continue
```

If QEMU is already waiting from `-S`, `continue` tells it to resume until a breakpoint is hit.

## Useful GDB commands for kernel debugging
- `info registers`: Show CPU register contents.
- `bt`: Print the current call stack.
- `thread apply all bt`: Backtrace all threads.
- `info threads`: List threads visible to GDB.
- `disassemble /m start_kernel`: Show both source and assembly for `start_kernel`.
- `list start_kernel`: Display source around the function.
- `print variable_name`: Inspect kernel variables or structures.
- `set $pc = start_kernel`: Jump execution to a new instruction address.
- `stepi`: Single-step one machine instruction.
- `continue`: Resume execution until the next breakpoint.

## Practical debugging examples
### Early boot debugging
Use `-S` to stop execution immediately on startup and break on early boot functions:

```gdb
(gdb) break start_kernel
(gdb) break setup_arch
(gdb) continue
```

This lets you inspect architecture setup, memory initialization, and early device probing.

### Panic and oops debugging
To catch a panic or kernel oops, set breakpoints and disable auto-reboot:

```gdb
(gdb) break panic
(gdb) break oops_end
(gdb) continue
```

When the panic triggers, inspect registers and the stack to determine the root cause.

### Debugging a driver or module
For subsystem debugging, break on driver entry points or allocation functions:

```gdb
(gdb) break my_driver_init
(gdb) break __kmalloc
(gdb) break do_IRQ
(gdb) continue
```

## Notes on build artifacts
- `bzImage`: Bootable compressed kernel image used by QEMU.
- `vmlinux`: Uncompressed ELF binary containing symbols and debug info.
- `System.map`: Symbol table mapping kernel addresses to symbol names.
- `.config`: The kernel build configuration file.

## References

- https://docs.kernel.org/process/index.html\#dealing-with-bugs
- https://www.youtube.com/watch\?v\=NDXYpR_m1CU
- https://www.youtube.com/watch\?v\=l3h7F9za_pc
- https://www.youtube.com/watch\?v\=7QiR3TOYajY
- https://people.cs.vt.edu/huaicheng/lkp-sp26/resources/debugging/\#2-gdb--qemu-debugging
- https://lkp.pierreolivier.eu/slides/12_Qemu.pdf
