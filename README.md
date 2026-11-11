# deck
Deck OS


# Deck Lite 0.2.9

An experimental x86-64 operating system with its own kernel and bootloader,
written in C and Assembly. It boots into a terminal, installs onto a virtual
IDE disk, and runs small ELF64 programs in user mode with memory protection.
It is not yet a complete Unix system and cannot run external packages.

## Getting started

Current image: `dist/Deck-Lite-0.2.9-amd64.iso`.

In VirtualBox, select Other/Unknown (64-bit), BIOS (EFI disabled), 128 MiB of
RAM, 1 CPU, and IDE PIIX4. Attach a new 32 MiB virtual disk as the primary
master and the ISO as the secondary master. To create a VM with these settings:

```sh
sh scripts/create-vm.sh Deck-Lite-029-nova
VBoxManage startvm Deck-Lite-029-nova
```

Log in as `root` with no password. The keyboard uses the US layout, without UTF-8.

The installed and validated `Deck-Lite-029` VM is already available in this
project: open `Abrir-Deck.desktop` or run `VBoxManage startvm Deck-Lite-029`.
`Abrir-Deck-028.desktop`, `Abrir-Deck-027.desktop`, `Abrir-Deck-026.desktop`,
`Abrir-Deck-025.desktop`, `Abrir-Deck-024.desktop`, `Abrir-Deck-023.desktop`,
`Abrir-Deck-022.desktop`, and `Abrir-Deck-011.desktop` provide access to
previous versions.

```text
help
hello
ucat /etc/hostname /etc/os-release
ucp /etc/hostname /root/hostname-copia
uexec /bin/ucat /root/hostname-copia
uspawn /bin/hello
childtest
waittest
sh
cd /etc
pwd
ls
ucat hostname
hello
exit
mem
iotest
ttyread
```

`hello` runs a C program in ring 3. `iotest` checks file reads and file
descriptors. `ttyread` reads and displays a typed line: the current time limit
is three seconds, including time spent waiting for input. `fault`,
`privileged`, `textwrite`, and `stackexec` deliberately trigger faults; the
terminal should remain functional. `spin` tests the execution time limit,
and `bss` tests data memory.

Built-in commands include `ls`, `cd`, `pwd`, `cat`, `echo`, `mkdir`, `touch`,
`rm`, `rmdir`, `cp`, `mv`, `chmod`, `stat`, `whoami`, `id`, `uname`, `hostname`,
`date`, `uptime`, `df`, `mount`, `sync`, `diskinfo`, `install`, and shutdown.
TAB completes commands, files, and directories, including paths containing
spaces. When multiple matches exist, it completes their common prefix and
displays the list. Quoting and redirection are supported in `echo`. Pipes are
available inside the ELF shell (`sh`); scripts are not yet supported.

## Installing to the virtual disk

Use only a disposable virtual disk. The installer uses the entire disk,
without partitioning or coexistence with other operating systems.

```text
diskinfo
install ada0
```

Check the disk shown and type `ERASE ada0`. Then run `shutdown -p now`,
remove the ISO, and boot from the disk. Run `sync` after editing files;
reboots and shutdowns also attempt to save changes. DeckFS maintains two
snapshots with CRC32, supporting up to 96 entries and 4096 bytes per file.
When booted from the ISO, files remain in RAM until installation. Existing
installations are not upgraded automatically: use a new VM to test this
version.

## Protected execution and limitations

The kernel checks a RAM pool against the E820 memory map, loads static ELF64
executables, and schedules up to four ring 3 processes. Code pages are
read-only, the stack is non-executable, and programs cannot access kernel
memory. Faults and timer events return control to the waiting parent or the
terminal; process resources are released. The global pool contains 128
physical pages shared among processes.

The custom ABI provides `write`, `exit`, `read`, `open`, `close`, `exec`,
`spawn_wait`, `read_wait`, `chdir`, `getcwd`, `readdir`, `complete`, `dup`,
`dup2`, `pipe`, `fork`, and `wait`. Files can be read, written, created,
truncated, or opened in append mode, with eleven file descriptors per
process, including streams 0, 1, and 2. `ucp source destination` copies
files in user mode. Run `sync` to persist changes to the installed disk.
Standard input supports non-blocking reads or waiting for a byte.
Executables receive arguments and can replace their process image with
another program using `exec`. A parent can start a child and wait for its
result. Background jobs, libc, and binary compatibility with Linux/FreeBSD
are not yet available.

`root` is the only identity, with no authentication. `chmod` changes metadata;
there is no multi-user support or corresponding access control. The main
terminal and built-in commands still run in the kernel. Run `sh` to open the
minimal ELF shell in ring 3; `exit` returns to the main terminal. This shell
supports help, status, cd, pwd, ls, TAB completion, program execution,
redirection (`<`, `>`, `>>`, `2>`, and `2>&1`), pipelines of up to three
commands, quoting, and backspace. Administrative commands remain in the
kernel terminal. Each process has its own working directory; input received
through `read_wait` renews the CPU budget for long-running sessions. Four
contexts are scheduled by the timer; pipes block without consuming CPU.

**`pkg install vim` does not work yet.** Networking, DNS, TLS, runtime
support, and package management are missing. UEFI, USB, AHCI, NVMe, SMP,
and a graphical interface are also unavailable. The ISO uses El Torito
floppy emulation and must not be written directly to a USB flash drive.
The `.img` image is experimental; prefer the ISO in VirtualBox.

## Building and testing

Required tools: Clang/LLD, llvm-objcopy, makefs, a shell, and utilities
compatible with the scripts. Integration uses VirtualBox.

```sh
make
make test
sh scripts/smoke-test.sh Deck-Lite-029-novo-teste
```

The integrated test creates a VM with a new disk, tests the ISO, installs
the system, and repeats the tests from the disk to verify persistence.
Choose a VM name that is not already in use. Logs are stored in `logs/`.
Previous VMs have been preserved.

Source code is in `src/boot`, `src/kernel`, and `src/user`; tools are in
`scripts/`; tests are in `tests/`; build artifacts are in `dist/`.
Reference downloads are not part of the build. The BSD-2-Clause license
is in `LICENSE`.

See [program execution and changes in 0.2.9](docs/ALTERACOES-0.2.9.md),
[changes in 0.2.0](docs/ALTERACOES-0.2.0.md), and the
[roadmap](docs/ROTEIRO.md).
