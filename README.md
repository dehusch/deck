# Deck Lite 0.4.2

Deck is an experimental x86-64 operating system with its own C/Assembly
kernel and BIOS loader. Boot starts init, password login and a protected
user shell. This release adds HTTPS downloads with TLS 1.2 certificate validation. External packages and Vim are not available.

## Start

Open `Abrir-Deck.desktop` to start the installed `Deck-Lite-042` VM.
Existing accounts, passwords and personal files are migrated from your 0.4.0
disk. The original 0.4.0 VM remains available through `Abrir-Deck-040.desktop`.
On a fresh installation, choose and confirm the root password (8–64 bytes).
Typing passwords is hidden. Initial root setup saves automatically on disk.

The keyboard defaults to German ISO/QWERTZ with UTF-8 accents and AltGr.
Use TAB for completion, Up/Down for history and Ctrl-C to cancel foreground
work. `keymap de` and `keymap us` switch layouts.

## Storage and file tools

DeckFS v3 supports **256 entries and 65536 bytes per file**. Directories also
consume entries. This is a bounded snapshot filesystem, not a general-purpose
block allocator; `df` reports usage. The VM disk must be at least 64 MiB.

```text
mkdir work
echo "Hello Deck" > work/hello
cp -r work backup
mv backup archive
ls -l archive
cat archive/hello
```

The copied file contains `Hello Deck`. `ls -l` shows permissions, owner,
primary group ID and size. Recursive copy validates the whole tree before
creating anything; permission, capacity or path failures leave no partial tree.
Directory moves update all descendant paths together. Files/directories in use
cannot be moved. Copying into a source subtree is rejected. Tree copies and
moves do not replace existing destinations; when the destination is a directory,
the source basename is appended. Regular `cp` can replace a writable file.

## Text editor

Run `edit notes.txt`. The editor is line-oriented:

| Command | Action |
| --- | --- |
| `a` | Append lines; a line containing only `.` ends insertion |
| `i N` | Insert before line N; finish with `.` |
| `d N` | Delete line N |
| `p` | Print numbered lines |
| `w` | Save the buffer into the filesystem in RAM |
| `q` | Quit; refuses when unsaved changes remain |
| `q!` | Discard unsaved changes and quit |

Example, after `edit notes.txt`:

```text
a
First line
Second line
.
p
w
q
```

`cat notes.txt` should show both lines. Ctrl-C cancels the current input;
previous buffer edits remain. Input lines are limited to 1023 bytes and files
to 64 KiB. Binary files containing NUL bytes are rejected. Saving validates
size and access before replacement, so a failed save leaves the old file intact.

**Persistence is explicit:** `w` saves in RAM. Run `sync` as root to commit
all filesystem changes to disk. Administrative `reboot` and `shutdown -p now`
also sync. Unsynced changes can be lost on a forced shutdown.

## Accounts and shell

As root, use `useradd NAME`, `passwd [NAME]`, `userdel NAME`, `users` and `sync`.
`exit` returns to login. `whoami`, `id` and `pwd` identify the session.
Homes use mode 0700; ordinary users cannot access `/root` or `/etc/shadow`.
Deleting an account retains its home and never reuses its UID.

The shell supports PATH, `export KEY=value`, `unset KEY`, `$KEY`, `${KEY}`,
`$?`, quotes, simple scripts, up to three pipeline stages and redirection.
It is not a POSIX shell: no loops, globbing, positional parameters or command
substitution. `rescue` opens the kernel console for interactive root sessions.

## Images, installation and upgrades

ISO: `dist/Deck-Lite-0.4.2-amd64.iso`. Use BIOS, 128 MiB RAM, one CPU,
IDE PIIX4 and a new 64 MiB virtual disk. The installer command is `install ada0`;
it requires typing `ERASE ada0`. Shut down, detach the ISO and boot the disk.
Changes on the uninstalled live ISO are volatile.

DeckFS v3 is not directly compatible with v2 snapshots. The offline migration
tool reads a copy of the v2 disk, verifies the latest valid snapshot and writes
a separate v3 disk. It retains accounts, homes, ownership, permissions and
custom files while refreshing known base programs. Never apply a raw format
conversion directly to a running VM or overwrite the source disk.

## Build and verify

Requires Clang/LLD, llvm-objcopy, makefs, Perl and VirtualBox for VM tests.

```sh
make
make test
sh scripts/test-session.sh Deck-Lite-042-session-check
```

Storage-specific VM regressions are in `scripts/test-storage-vm.pl`; they use
migrated disposable fixture accounts, not real credentials. They check large
files, full filesystem handling, directory operations, the editor, permissions
and reboot persistence. See [release notes](docs/RELEASE-0.4.2.md) for evidence
and [the English test guide](docs/TESTING-0.3.1.md) for a repeatable procedure.

Limits still include eight accounts, eight task slots, no supplementary
groups, no symlinks, no general VFS/mount support, no UEFI/USB, and no
external OS binary ABI. Password hashing uses PBKDF2-HMAC-SHA256; this prototype
has not undergone a security audit. Historical Portuguese documentation remains
unchanged; new documentation and system text are written in English.

## Text processing (0.3.2)

Use `grep`, `head`, `tail`, `wc`, `sort`, `uniq` and `tee` with files and pipes:

```text
sort words | uniq -c | cat
grep -n apple words
cat words | tee copied | wc -l
```

Input is bounded to 64 KiB; sort/uniq additionally allow 4096 lines. Grep uses
literal matching, sorting is bytewise, and wc counts UTF-8 bytes. These are
prototype tools, not full POSIX replacements. See
[the 0.3.2 release notes](docs/RELEASE-0.3.2.md) for options and acceptance tests.

## Networking and background jobs (0.4.0)

The installed Deck-Lite-042 VM uses Intel 82540EM bridged to re0. DHCP
configures an address on the router's LAN automatically. The portable VM
creation helper still defaults to NAT.

```text
ifconfig
ping -c 3 www.google.com
nslookup example.com
sleep 30 &
jobs
ps
fg
```

Ctrl-C cancels the foreground task. `kill PID` terminates an owned process;
`wait [PID]` waits for background jobs. Logout cleans up session tasks.
`dhclient` restarts DHCP as root. External ping by IP and Google hostname passed after switching the installed
VM from NAT to bridge mode. The NAT path on this host still times out. See the
[0.4.0 release notes](docs/RELEASE-0.4.0.md) for limits and test procedures.

## HTTP downloads (0.4.1)

```text
fetch -o example.html http://example.com/
grep Example example.html
fetch -o - http://example.com/ | wc -c
fetch -o background.html http://example.com/ &
wait
```

Fetch supports HTTP, explicit ports, Content-Length, chunked responses and
orderly EOF. Downloads are limited to 65536 bytes. A destination passed with
`-o FILE` is replaced only after a complete valid response; failures and Ctrl-C
preserve an existing file. File permissions and explicit disk sync still apply.
HTTPS is supported with certificate validation; redirects and compression are not implemented. See the
[0.4.2 release notes](docs/RELEASE-0.4.2.md) for socket APIs, limits and tests.

## HTTPS downloads (0.4.2)

```text
fetch -o example.html https://example.com/
grep Example example.html
fetch -o - https://example.com/ | wc -c
```

HTTPS checks the certificate chain, hostname and validity dates. It requires a
UTC hardware clock and RDRAND support; if either is unavailable it refuses the
connection. Root-owned `/etc/tls/roots.bin` contains 15 selected public roots.
There is no insecure mode or plaintext fallback. Certificate errors preserve
an existing destination when using `-o FILE`.

This bounded TLS 1.2 client has no TLS 1.3, revocation checks, IP-address
certificate identities or full browser PKI policies. Some sites are unsupported.
See [the release notes](docs/RELEASE-0.4.2.md) for protocol, trust and test details.
