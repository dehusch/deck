# Deck Architecture and Design Decisions

## Objective

Create Deck as a custom operating system written in C/Assembly, with a Unix environment, **its own kernel**, and the long-term goal of running external applications. The product identity is exclusively Deck.

Version 0.2.8 adds protected execution and file access for programs. A familiar directory structure and commands with matching names do not guarantee POSIX compliance or compatibility with external binaries.

## Current Decisions

| Area | Implemented decision | Rationale / consequence |
| --- | --- | --- |
| CPU | AMD64 / x86-64, single CPU | Common target in VirtualBox. i386 requires a separate target and image. |
| Languages | Freestanding C11 and x86 Assembly | No runtime from another OS; host tools are not included in the image. |
| Kernel | Minimal monolithic kernel | Drivers and initial terminal in the kernel; optional ELF shell; one ELF64 process in ring 3 at a time. |
| Boot | Custom 512-byte BIOS bootloader | Enters long mode without GRUB, Limine, or an external kernel. UEFI is a separate stage. |
| ISO | El Torito, 1.44 MiB floppy emulation | BIOS reads the boot sector, which reads 256 sectors reserved for the kernel. |
| Memory | Identity mapping of the first GiB using 2 MiB pages | Initial mapping; this neither allocates nor detects 1 GiB of RAM. |
| Console | VGA 80×25, US PS/2, COM1 | Suitable for a BIOS VM, without modern graphics support. |
| Identity | Root UID 0, wheel GID 0 | No authentication or ordinary users in this version. |
| Storage | DeckFS, RAM + snapshots via ATA PIO | Small, independent implementation. Not equivalent to UFS/ZFS. |
| Installation | Entire primary IDE disk, explicit confirmation | Does not allow selection of other disks or partitions. A new virtual disk is recommended. |
| Packages | Not implemented | The infrastructure required by external executables is missing. |

## Memory and Boot

- `0x0500`: BIOS drive used for booting (passed to the kernel).
- `0x1000..0x3fff`: PML4, PDPT, and PD tables.
- `0x7c00`: BIOS bootloader.
- `0x5000`: E820 count/signature; map entries starting at `0x5010`.
- `0x10000`: BIOS E820 query; long-mode entry at `0x10200`. Text, constants, and data use a reserved 131072-byte region.
- `0x200000`: start of BSS. The linker rejects BSS that reaches the area reserved for installation.
- `0x3ff000`: bootloader copy for installation.
- `0x400000`: copy of the 256 loaded sectors, preserved before modifying kernel data.
- `0x800000`: initial top of the stack.

The bootloader reads the kernel via BIOS CHS from a floppy-emulated CD or via EDD/LBA from disk. It enables A20, GDT, PAE, EFER.LME, and paging, then enters the 64-bit kernel. The Assembly entry point preserves the original image, clears BSS, and calls `kmain()`.

The C code configures GDT/TSS/IDT, console, PIT, filesystem, and ATA. Interrupts remain disabled in the kernel except during read_wait; the user process enables the timer, which limits its execution to approximately three seconds. Keyboard, serial, and disk use polling. User faults terminate the process with diagnostics and return control to the waiting parent or shell; kernel faults halt the CPU.

A pool of 128 pages is selected from RAM verified by the E820 map, starting at 16 MiB. The process has mappings at 0x40000000, with write-protected code, an NX stack, and kernel memory restricted to supervisor mode. The custom ABI provides write, exit, read, open, close, exec, spawn_wait, and read_wait; argc/argv are placed on the user stack. Descriptors enforce read/write modes and are released on exit.

Exec prepares the new image in separate pages and only switches mappings after complete success. Errors discard the prepared image and preserve the caller. The switch rewrites the syscall return frame while retaining the PID and already-consumed time budget. Descriptors are inherited unless marked DECK_O_CLOEXEC. Spawn_wait starts child processes with the parent blocked.

The kernel does not use BIOS services after switching to long mode. The installed disk driver is ATA PIO LBA28, primary master. Devices must match the documented configuration.

## DeckFS v1 Format

The disk has no conventional partition table. LBA 0 contains the bootloader; LBAs 1–256 hold the kernel reservation. The two snapshot areas begin at LBAs 2048 and 4096, each with space for 1024 sectors.

The first sector of each snapshot contains `DECKFS1`, version, size, sequence number, and payload CRC32. The payload is the filesystem structure: a 64-byte hostname and 96 entries. Each entry stores type, mode, UID, GID, size, a 160-byte path, and 4096 bytes of content. All integer fields use the target's little-endian layout; the format version must change if the layout changes.

On save, the kernel writes the payload to the alternate slot, requests a flush, writes the header last, and requests another flush. On boot, it accepts only snapshots with valid metadata and a correct checksum, choosing the newest sequence. An interrupted write can lose recent changes; the other slot preserves the previous version if it is still intact. The payload CRC detects accidental corruption, without cryptographic authentication. The header has no checksum of its own; this limits robustness to simple failure scenarios.

Files are edited in RAM. `sync` is explicit, with an automatic attempt before shutdown/reboot. There is no full journaling, effective permissions, timestamps, links, special files, or consistency across concurrent processes.

## Decisions for Future Versions

1. **Processes and system calls:** expand directory APIs and the userland shell; maintain isolation and memory validation.
2. **External binary compatibility:** recommendation: create a separate, testable ABI layer; do not spoof `uname`/ABI to download packages before being able to execute them.
3. **Final filesystem:** recommendation: implement VFS first, then decide whether to implement UFS2 or adopt a compatible, properly licensed implementation. DeckFS is only for the prototype. ZFS adds complexity and higher requirements.
4. **Modern boot:** recommendation: add a custom UEFI loader in C while keeping the kernel independent. UEFI firmware is not an external kernel.
5. **Security:** isolated processes, credentials, permissions, and authentication before enabling network services or storing real data.
6. **Updates and packages:** native Deck packages before external binary compatibility could reduce dependencies; the goal of running `pkg install vim` directly against the desired external repository still requires a compatible ABI and libraries.
7. **32-bit support:** defer i386 until amd64 is stable. Supporting i386 binaries on an amd64 CPU and booting on a 32-bit-only CPU are different tasks.

## Engineering References

- [Intel SDM](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html): execution modes, paging, descriptors, and x86 architecture.
- `/usr/src/usr.sbin/makefs/cd9660/cd9660_eltorito.c` on the host: examined to correct the initial sector count in the El Torito catalog. The custom `scripts/isofix.c` tool validates the catalog and sets this field to 1.

In 0.2.8, spawn_wait saves the parent's frame, pages, descriptors, and time
budget. The child uses different physical pages within the same virtual range.
A single kernel entry stack serves both without reentrancy: only the return
frame is replaced, without suspending a C call inside the kernel.
On termination, the child releases its pages and restores the parent. There
are at most four contexts, chained together, without concurrent execution.
The 128-page pool is global.

In 0.2.8, there are four chained contexts and 128 pages in the global pool.
The ELF shell `/bin/sh` is started manually. read_wait waits for a byte in
ring 0 with interrupts enabled and HLT; the ring 0 timer does not consume the
user time budget. No other process is scheduled during the wait. The kernel's
initial terminal remains available for administration and TAB completion.

In 0.2.8, the current directory follows the context: spawn copies it, exec
preserves it, and exit restores the parent or initial terminal. Chdir/getcwd/
readdir validate pointers and sizes at the ABI boundary. The ELF shell offers
cd/pwd/ls and TAB; complete reuses the kernel's candidate search with the
ELF shell's command list. Read_wait replenishes the user time budget when
actual input is received.

In 0.2.8, descriptors 0..10 point to file descriptions. Dup/dup2 share the
file offset; spawn copies the descriptor table and shares file descriptions
with the parent, including offsets and access flags. CLOEXEC is a property
of the descriptor. The shell saves affected streams with CLOEXEC, applies
redirections in order, and restores the streams after execution. Read_wait
reads files until EOF when stdin is redirected; only actual terminal input
replenishes the CPU time budget.

In 0.2.9, the parent chain was replaced with four process slots, with states
for runnable, waiting for a child, reading/writing a pipe, terminal, and zombie.
The timer alternates between runnable processes. Blocked calls save the frame
and repeat int 0x80 when the condition allows progress; pointers are validated
again. Fork copies physical pages; exec and spawn continue preparing the image
before committing it. The active directory and page table are switched along
with the frame. Pipes have a 1024-byte circular buffer and reference counts
in the same global file descriptions used by files. See ALTERACOES-0.2.9.md
for limits, EOF handling, descendant termination, and deadlock recovery.

In 0.2.10, keyboard.h separates German scancode translation, composition,
and the UTF-8 queue. The timer and syscalls detect Ctrl-C even when the user
runs a loop that does not read input. The group corresponds to the process
tree beneath the nearest interactive shell; termination keeps direct-child
zombies for wait and releases resources immediately. Creation of new children
is blocked until the next prompt after cancellation, preventing a race
condition during fork.

History.h provides two volatile histories of 16 lines each. The shared line
editor handles UTF-8, TAB, and arrow keys; syscall 18 checks the buffer,
terminal, and absence of other active processes before serving it. Shell
logic remains in ring 3. Unicode.h decodes VGA output and installs additional
Latin/symbol glyphs; serial output receives UTF-8 without conversion. Access
to the VGA font follows the registers documented in the local driver
/usr/src/sys/dev/fb/vga.c; the code and glyphs added to Deck are original.

## Session and Accounts in 0.3.0

Boot starts init/login/sh in ring 3. UID/GID and 16 environment variables are
part of each task; exec preserves them, while fork/spawn inherit them.
Authenticated login closes extra descriptors, resets the environment/cwd,
and clears the history. Syscalls 19..23 add environment management, limited
administration, filesystem operations, accounts, and no-echo input. The kernel
console requires root authorization in the interactive session and is used
for recovery.

DeckFS v2: 120 nodes, 8192 bytes per file, snapshots at LBA 2048/4096,
CRC validation; it does not migrate the previous layout. The kernel has
a 384-sector reservation and BSS at 5 MiB, below the stack at 8 MiB.
The scheduler supports 8 tasks, 88 file descriptions, 8 pipes, and a pool
of 256 pages.

`permissions.h`, `fsops.h`, `accounts.h`, and `password.h` handle permissions,
operations, the account database, and password derivation. The base system
uses multicall executables with dispatch by basename; the supported names
are those installed in /bin. An unknown name fails without performing file
operations.

`scripts/seed-disk.c` uses the same seeds as the kernel to create a clean
installed disk, without predefined credentials; this prepares the delivery
VM. Booting that disk was verified through initial root configuration.
