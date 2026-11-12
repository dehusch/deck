# Roadmap for a Custom Unix System

## 0.1 — Bootable Terminal (implemented)

BIOS boot, custom x86-64 kernel, console and keyboard, built-in commands, Unix directories, root identity, IDE installer, and experimental persistence. This release does not fulfill the goal of a complete base system.

## 0.2 — Kernel Foundation

- E820/UEFI RAM map, page allocator and heap, page and stack protection.
- Exception handling with context and diagnostics, interrupts, timer, and scheduler.
- Ring 0/ring 3 transition, per-process address space, static ELF64.
- Minimal system calls: read/write, open/close, exit, wait, and process creation/execution.
- Acceptance test: a C userland program writes to the terminal and exits; invalid memory access terminates the process without crashing the kernel.

## 0.3 — Unix Base System

- `init` and shell as actual programs, minimal libc, descriptors, pipes, general redirection, and signals.
- `fork`, `execve`, `waitpid`, `mmap`, `ioctl`, terminals, and the required set of syscalls.
- VFS, a robust persistent filesystem, and mounting pseudo-filesystems at `/dev` and `/proc` if chosen.
- User accounts and groups, effective credentials, permissions, login, and proper password-hash storage.
- Separate executables for utilities, manual pages, and Unix behavior tests.
- Acceptance test: boot invokes `/sbin/init`, presents a login prompt, and runs `/bin/sh`; an ordinary user cannot modify protected files.

## 0.4 — Devices and Networking

UEFI, ACPI, PCI, AHCI or virtio-blk, a network driver suitable for the VM, Ethernet, ARP, IPv4, ICMP, UDP/TCP, DHCP, and DNS; USB and NVMe later, depending on the target hardware. TLS requires a cryptographic library, reliable entropy, a clock, and certificate validation.

Acceptance test: obtain an IP address via DHCP and download a file over HTTPS with certificate validation.

## 0.5 — External Binary Compatibility

Package compatibility does not come from the directory tree. Implement and test:

- The reference system's amd64 ELF ABI, ELF notes, initial stack, auxv, TLS, dynamic linker, and calling conventions.
- Sufficient system calls and return-value/errno behavior, signals, threads, synchronization, sockets, polling/kqueue, and filesystem operations.
- Compatible libraries and the expected runtime; check licenses and preserve notices when incorporating third-party code.
- ABI identification for `pkg`, package catalog, dependencies, repository signatures, extraction, post-installation scripts, and package database.
- Version compatibility: packages may depend on features of specific external kernels, including modules, which do not automatically become functional.

Start with a small static binary. Then move on to dynamic ELF, libc, shell, and simple applications. Only then validate a real `pkg` and Vim.

**Final acceptance criterion requested by the user:** on a clean installation with networking, `pkg install vim` accesses a compatible external repository, validates and installs the package and its dependencies; `vim` launches and edits a file that persists after reboot. This criterion has not yet been met.

## Progression Toward Real-World Use

Testing across multiple virtual machines and physical hardware, failure recovery, kernel and base-system updates, an installer with partitioning, and security, compatibility, and performance documentation. No short estimate for a “complete Unix” is assumed: this is an ongoing engineering project.

## Status in 0.2.9

Completed in the prototype: protected ELF execution, file reads/writes, exec,
spawn_wait, an ELF shell with directories and TAB completion, dup/dup2,
redirections, and pipelines of up to three stages with fork/wait and scheduling.
The proposed next step is Ctrl-C handling for the foreground group. The ultimate
goal of package support and Vim still depends on the networking, runtime, and
compatibility stages described above.

## Status in 0.2.10

German keyboard/UTF-8 support, terminal history and line editing, and foreground
group interruption were delivered together. Related tasks should be grouped
into usable milestones without requiring confirmation between improvements to
the same terminal. The next proposal combines the standard userland session
with its core utilities/environment; details are in PROXIMO-MARCO.md.

## Status in 0.3.0

Boot via init/login/sh, password-protected accounts, per-process UID/GID,
permissions, external utilities, environment/PATH, and simple scripts are
implemented. The session and system-file protection criteria were validated
in the VM. This does not complete all broader items in stage 0.3: VFS,
mmap/ioctl, POSIX signals, libc, and full terminal support are still pending.
Networking/pkg/Vim are also pending.
