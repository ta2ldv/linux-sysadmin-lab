<div align="right">

**English** | [Türkçe](README.tr.md)

</div>

# Linux Sysadmin Lab

A hands-on Linux system administration lab: systemd, users & groups, apt, logging, processes, SSH and networking — learning how a Linux system actually works by poking at it, not by reading about it.

All experiments run on a real **Ubuntu** machine, and the outputs shown are real, not idealized.

The longer-term goal behind this lab is virtualization and Kubernetes. Almost every chapter here is a building block for that world: systemd runs the kubelet, users and permissions become security contexts, veth pairs and bridges are how pods talk to each other. This lab is the Linux underneath Kubernetes.

## Table of contents

- [Part 0 — Linux filesystem layout](#part-0--linux-filesystem-layout)
- [Part 1 — Everything is a file](#part-1--everything-is-a-file)
- [Part 2 — systemd & systemctl](#part-2--systemd--systemctl)
- [Part 3 — Logging & journalctl](#part-3--logging--journalctl)
- [Part 4 — Users & groups](#part-4--users--groups)
- [Part 5 — Package management (apt)](#part-5--package-management-apt)
- [Part 6 — Process management](#part-6--process-management)
- [Part 7 — SSH & sshd](#part-7--ssh--sshd)
- [Part 8 — Networking basics](#part-8--networking-basics)
- [Part 9 — File permissions & ownership](#part-9--file-permissions--ownership)
- [Part 10 — Disk & filesystem](#part-10--disk--filesystem)
- [Part 11 — Cron & timers](#part-11--cron--timers)

## Curriculum

| # | Part | Question it answers | Status |
|---|------|--------------------|--------|
| 0 | [Linux filesystem layout](#part-0--linux-filesystem-layout) | What lives where in the directory tree — what are /etc, /var, /usr, /bin for? | ✅ |
| 1 | [Everything is a file](#part-1--everything-is-a-file) | What is a file, a file descriptor, a socket — and why is *everything* one? | ⏳ |
| 2 | [systemd & systemctl](#part-2--systemd--systemctl) | How does systemd control every program on the machine? | ✅ |
| 3 | [Logging & journalctl](#part-3--logging--journalctl) | Where do logs live, and how do I interrogate the journal? | ✅ |
| 4 | [Users & groups](#part-4--users--groups) | How do I create, restrict and destroy users — and what is a group really? | ✅ |
| 5 | [Process management](#part-5--process-management) | What is a process, a signal — and what really separates SIGTERM from SIGKILL? | 🔜 |
| 6 | [Package management (apt)](#part-6--package-management-apt) | What actually happens on `apt install` — repos, GPG keys, binaries? | 🔜 |
| 7 | [SSH & sshd](#part-7--ssh--sshd) | How do I set up and harden sshd, and manage keys properly? | 🔜 |
| 8 | [Networking basics](#part-8--networking-basics) | How does the machine talk — interfaces, DNS, firewall, namespaces? | 🔜 |
| 9 | [File permissions & ownership](#part-9--file-permissions--ownership) | Who may touch what — chmod, umask, setuid, ACL? | 🔜 |
| 10 | [Disk & filesystem](#part-10--disk--filesystem) | How do disks become directories — mount, fstab, LVM? | 🔜 |
| 11 | [Cron & timers](#part-11--cron--timers) | How do I run things on a schedule — and cron vs systemd timers? | 🔜 |
| 12 | [Memory & I/O](#part-12--memory--io) | How is memory managed — what do page cache, dirty pages, writeback, fsync actually do? | 🔜 |

---

# Part 0 — Linux filesystem layout

Machine: Ubuntu 24.04 (AWS), architecture amd64 (verified — see 0.9). This part isn't lab practice, it's a standard FHS (Filesystem Hierarchy Standard) reference.

## Cheat sheet

| Directory | Holds | Persistent | Written by |
|-----------|-------|------------|------------|
| `/etc` | system-wide config files (text) + override layer | persistent | root, packages at install time |
| `/boot` | kernel image, initramfs, bootloader config | persistent | kernel package, `update-grub` |
| `/var` | data that changes: logs, cache, spool, databases | persistent | services, root |
| `/usr` | installed programs + libraries + shared data | persistent, apt-managed | package manager (apt) |
| `/usr/bin` | commands anyone (a normal user) runs | persistent, apt-managed | package manager |
| `/usr/sbin` | commands the admin/root runs | persistent, apt-managed | package manager |
| `/usr/lib` | libraries + programs' own internal binaries (not just `.so` files) | persistent, apt-managed | package manager |
| `/bin` | core commands (`ls`, `cat`...) — symlink to `/usr/bin` on Ubuntu | persistent | package manager |
| `/home` | users' personal files | persistent | the user themself |
| `/tmp` | short-lived temporary files | cleaned periodically (`systemd-tmpfiles`), tmpfs or disk depending on the distro (on this machine: disk — see 0.4) | everyone (world-writable, sticky bit) |
| `/opt` | non-apt, self-contained 3rd-party software | persistent | manual installs |
| `/proc` | running processes + kernel state — not a real file, a live view of the kernel | in RAM, not on disk | kernel |
| `/sys` | virtual fs where the kernel exports device/driver info | in RAM, not on disk | kernel |
| `/dev` | device nodes (`/dev/sda`, `/dev/null`, `/dev/tty1`...) | in RAM (devtmpfs), not on disk | kernel (udev) |
| `/mnt` | admin's manual/temporary mount point | persistent (empty unless something is mounted) | admin, cloud-init (instance store) |
| `/media` | auto-mount point for removable media (USB/CD) | persistent (usually empty on a headless server) | udisks2 (auto-mount) |
| `/lib` | kernel modules + shared libraries for core programs — symlink to `/usr/lib` on Ubuntu | persistent | package manager |
| `/lib64` | fixed path for the 64-bit ELF interpreter — symlink to `/usr/lib64` | persistent | package manager (usrmerge) |

## 0.1 — Directory tree (overview)

```
/ ─┬─ bin/ -> usr/bin              # symlink (usrmerge)
   ├─ sbin/ -> usr/sbin            # symlink (usrmerge)
   ├─ lib/ -> usr/lib              # symlink (usrmerge)
   ├─ lib64/ -> usr/lib64          # symlink, fixed path for the 64-bit ELF interpreter -- see 0.9
   ├─ boot/                        # kernel image, initramfs, bootloader -- see 0.10
   ├─ etc/                         # system-wide config, text + override layer -- see 0.6
   ├─ usr/ ─┬─ bin/                # commands anyone runs -- see 0.7
   │        ├─ sbin/               # commands the admin runs -- see 0.7
   │        └─ lib/                # libraries + the program's own internal binaries -- see 0.7
   ├─ var/ ─┬─ log/                # log files
   │        ├─ lib/                # service state/data, persistent
   │        └─ tmp/                # temp files kept longer than /tmp, always on disk
   ├─ tmp/                         # short-lived temp files -- on this machine: disk, not tmpfs, see 0.4
   ├─ opt/                         # non-apt, 3rd-party software
   ├─ home/                        # user home directories
   ├─ root/                        # root user's home, separate from /home
   ├─ proc/                        # virtual, kernel process view -- not on disk
   ├─ sys/                         # virtual, kernel device/driver view -- not on disk
   ├─ dev/                         # device nodes (devtmpfs) -- not on disk
   ├─ mnt/                         # admin's manual mount point -- see 0.8
   ├─ media/                       # removable media auto-mount point -- see 0.8
   └─ run/                         # tmpfs, runtime data -- recreated from scratch on every boot
```

## 0.2 — Why `/bin` and `/lib` are symlinks

Ubuntu adopted "usrmerge": `/bin`, `/sbin`, `/lib` used to be separate directories at the root, because early boot might happen before `/usr` was mounted. Now that the initramfs mounts everything early, the split became pointless — everything moved under `/usr`, and the old names stayed as symlinks for backward compatibility.

```
$ ls -la /
lrwxrwxrwx ... bin -> usr/bin
lrwxrwxrwx ... lib -> usr/lib
lrwxrwxrwx ... sbin -> usr/sbin
```

| Path | Real location |
|------|----------------|
| `/bin/ls` | `/usr/bin/ls` |
| `/sbin/reboot` | `/usr/sbin/reboot` |
| `/lib/systemd` | `/usr/lib/systemd` |

## 0.3 — `/proc`, `/sys`, `/dev`: not on disk

All three are **virtual** filesystems. They show up in `df -h` but use no disk space; the kernel recreates them from scratch on every reboot.

| Directory | Shows | Example |
|-----------|-------|---------|
| `/proc/<pid>/` | state of that process | `/proc/1/status`, `/proc/1/cmdline` |
| `/proc/cpuinfo`, `/proc/meminfo` | kernel's view of hardware/resources | `cat /proc/meminfo` |
| `/sys/class/net/` | kernel objects for network interfaces | `/sys/class/net/eth0` |
| `/dev/sda`, `/dev/null`, `/dev/tty1` | device node — writing to it means talking to hardware | `echo hi > /dev/null` |

## 0.4 — `/tmp` vs `/var/tmp` vs `/opt`

| Directory | Lifetime | Use |
|-----------|----------|-----|
| `/tmp` | short — periodically swept by `systemd-tmpfiles` | short-lived temp files |
| `/var/tmp` | kept longer than `/tmp`, always on disk | temp files for long-running jobs |
| `/opt` | persistent, not managed by apt | 3rd-party software that ships as a single self-contained package (e.g. `/opt/google/chrome`) |

**Verified on this machine: `/tmp` is disk, not tmpfs.**

```
$ mount | grep " /tmp "
(empty -- no match, not a separate mount point)
$ df -h /tmp
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        19G  2.6G   16G  15% /
$ systemctl cat tmp.mount
No files found for tmp.mount.
```

On Ubuntu 24.04, systemd's `tmp.mount` unit is shipped by the package to `/usr/share/systemd/tmp.mount`, not `/usr/lib/systemd/system/` — meaning the unit is never even "installed" (not disabled, just absent). With no `/tmp` entry in `/etc/fstab` either, `/tmp` stays a plain directory under `/` (root filesystem, disk). This isn't specific to the cloud image, it's Ubuntu 24.04's general default (Debian 13/trixie switched it to tmpfs, Ubuntu 24.04 did not). `systemd-tmpfiles` has nothing to do with this — it only cleans up old files inside `/tmp` (older than 10 days on 24.04), it has no bearing on the mount itself.

Sources:
- https://packages.ubuntu.com/noble/amd64/systemd/filelist
- https://www.debian.org/releases/trixie/release-notes/issues.en.html
- https://github.com/systemd/systemd/blob/main/units/tmp.mount

## 0.5 — The big picture: 3 principles that tie the system together

Instead of memorizing these directories as a flat list, it helps to see them as answers to "why is it split this way" — 3 principles, each detailed in its own subsection below.

| Principle | Summary | Detail |
|---|---|---|
| `/etc` = the override layer | `/usr` = the software exactly as it shipped, `/etc` = the part you customized for this machine | 0.6 |
| The split inside `/usr` = who runs it | `bin` = everyone, `sbin` = the admin, `lib` = the program itself | 0.7 |
| `/dev` ↔ mount points | `/dev/sdb1` is the hardware itself, `/mnt`/`/media` are empty doors opened onto that hardware | 0.8 |

## 0.6 — `/etc`: the override layer

The `/usr/lib` (vendor default) vs `/etc` (admin override) logic seen in Part 2 (systemd) is actually a pattern generalized across all of Linux: `/usr` = stays exactly as the package shipped it, `/etc` = the change you made for this specific machine. The `<package>.d/` drop-in pattern is the concrete, repeating form of this: `apt.conf.d`, `sudoers.d`, `cron.d`, `journald.conf.d` (Part 3), systemd unit overrides (Part 2) — all the same idea.

| Pattern | Purpose | Example |
|---|---|---|
| `/etc/<x>.d/` drop-in | add something without touching the package's own file | `/etc/systemd/journald.conf.d/99-lab.conf` |
| `/etc/default/<service>` | Debian-specific, `KEY=value` shell-format startup parameter file (not config) — sourced by the service's init script/unit at start | `/etc/default/grub`, `/etc/default/useradd` |
| `/etc/init.d/` + `/etc/rc?.d/` | SysV leftovers; a script with no native systemd unit gets turned into a virtual unit at boot by `systemd-sysv-generator` (`systemctl status` shows "Loaded: ... generated") | still 15-20 scripts on 24.04 |

Exception: the one exception to "`/etc` is always text" is `/etc/ld.so.cache` — that one's binary.

Source: FHS 3.0 §3.7, `man 8 systemd-sysv-generator`

## 0.7 — The split inside `/usr`: who runs it, for whom

| Directory | Who runs it | Example |
|---|---|---|
| `/usr/bin` | everyone (a normal user) | `ls`, `cat` |
| `/usr/sbin` | admin/root ("system binaries") | `useradd`, `sshd`, `reboot` |
| `/usr/lib` | the program itself (libraries + internal binaries, not just `.so` files) | `*.so` files, `/usr/lib/systemd/systemd`, `/usr/lib/apt/methods/` |

The "s" in "sbin" = system. The split historically meant "should be on root's PATH, not a normal user's" — in practice that's eroded today (Ubuntu puts both on PATH, Fedora 42 merged bin/sbin), but the naming logic is still this.

Source: FHS 3.0 §4.4-4.7, `man 7 hier`

## 0.8 — `/dev` ↔ mount points

- Something like `/dev/sdb1` is the hardware ITSELF — the kernel's raw interface saying "this disk is here"; reading/writing it means talking directly to the hardware.
- Directories like `/mnt`, `/media` are EMPTY DOORS — once you "mount" the filesystem inside a device (`/dev/sdb1`) onto this door, you can browse the disk's contents like a normal folder.
- So: `/dev` = hardware, `/mnt`/`/media` = the window opened onto that hardware's contents. `mount /dev/sdb1 /mnt/usb` connects the two.

| Directory | Who mounts it | When |
|---|---|---|
| `/mnt` | admin, manually/temporarily | on AWS, if an instance store (ephemeral disk) exists, cloud-init mounts it here by default |
| `/media` | udisks2, automatically | removable media (USB/CD), usually `/media/<user>/<label>` — added to FHS in 2004 (FHS 2.3) because `/mnt` was being used for both cases interchangeably |

On a server (like ours, headless), `/media` usually stays EMPTY — there's no desktop auto-mount scenario. `/mnt` is still actively used on AWS.

Source: FHS 3.0 §3.11-3.12, cloud-init docs (mounts module)

## 0.9 — `/lib32` and `/lib64`: live verification

This machine is amd64:

```
$ dpkg --print-architecture
amd64
$ ls -la /lib64 /lib32
ls: cannot access '/lib32': No such file or directory
lrwxrwxrwx 1 root root 9 Apr 22  2024 /lib64 -> usr/lib64
```

Ubuntu (the Debian family) uses "multiarch" — libraries live in triplet directories: `/usr/lib/x86_64-linux-gnu/`. `/lib64` still exists on every amd64 system, but holds exactly ONE thing: `ld-linux-x86-64.so.2` (a symlink) — because the x86-64 ABI hardcodes the interpreter path into every 64-bit ELF binary as the FIXED `/lib64/ld-linux-x86-64.so.2`; without that path, no binary runs.

`/lib32` only appears once the legacy `libc6-i386` (32-bit compatibility) package is installed; it's not installed on our machine, so it's absent (normal, expected).

| Architecture | `/lib64` | Why |
|---|---|---|
| amd64 (this machine) | present, symlink → `/usr/lib64` | the ABI requires a fixed interpreter path |
| arm64 | may not exist | uses a different interpreter path |

Source: FHS 3.0 §3.10/§4.8, https://wiki.debian.org/Multiarch/Implementation

## 0.10 — `/boot` (Ubuntu 24.04, AWS)

| File | What |
|---|---|
| `vmlinuz-*` | compressed kernel image (the `linux-aws` flavor on AWS) |
| `initrd.img-*` | initramfs — the "early userspace" that runs BEFORE the root fs is mounted, loads disk drivers (nvme/ena), finds `/`, and `switch_root`s into it (the concrete counterpart of "initramfs mounts `/usr` early" from 0.2) |
| `config-*` | the kernel's build options |
| `System.map-*` | kernel symbol table (for crash/debug) |
| `vmlinuz`, `initrd.img`, `*.old` | symlinks to the latest/previous kernel |
| `grub/grub.cfg` | the bootloader menu — a GENERATED file, never hand-edited; produced by `update-grub` from `/etc/default/grub` + `/etc/grub.d/` |
| `/boot/efi` | UEFI ESP (FAT) mount point (empty if the instance boots BIOS-style) |

Source: `man 8 update-grub`, `man 7 bootup`, FHS 3.0 §3.5

## Notes

- `/usr` is apt-managed territory: everything you `apt install` lands here, hands off otherwise.
- Service accounts' home is usually `/nonexistent` or `/var/lib/<service>`, not under `/home` (see Part 4).
- `/etc/ld.so.cache` — the one known exception to "`/etc` is always text", it's binary itself.
- `grub.cfg` is never hand-edited — `update-grub` overwrites it every time it runs; make changes via `/etc/default/grub` or `/etc/grub.d/` instead.
- `/lib32` being absent on this machine is normal and expected: the `libc6-i386` package isn't installed.

[↑ Go back to TOC](#table-of-contents)

---

# Part 1 — Everything is a file

> ⏳ Placeholder — files, inodes, file descriptors, special files (device, pipe, socket), `lsof`, `/proc/PID/fd`.

[↑ Go back to TOC](#table-of-contents)

---

# Part 2 — systemd & systemctl

Machine: Ubuntu 24.04 (AWS), account `ubuntu` (sudo).

## Cheat sheet

| Command | What it does |
|---|---|
| `ps -p 1 -o pid,comm` | show who PID 1 is (systemd) |
| `ps -ef --forest` | draw the process tree with indentation, show parent/child relationships |
| `ps -p <pid> -o pid,ppid` | show a process's parent PID |
| `systemctl status <service>` | show status |
| `sudo systemctl start <service>` | start now |
| `sudo systemctl stop <service>` | stop now |
| `sudo systemctl restart <service>` | stop + start |
| `sudo systemctl reload <service>` | re-read config without killing the process (services that support it) |
| `sudo systemctl enable <service>` | start automatically at boot (creates a symlink) |
| `sudo systemctl disable <service>` | don't start at boot (removes the symlink) |
| `journalctl -u <service> -n N` | last N log lines |
| `journalctl -u <service> -f` | follow the log live |
| `journalctl -u <service> -p err` | only error+ severity logs |
| `ls -la /etc/systemd/system/` | units the admin added/enabled by hand, and symlinks |
| `sudo systemctl daemon-reload` | tell systemd about a change to an existing unit file |
| `systemctl show <unit> -p <Prop>` | show the final/effective value of a single property |
| `systemctl cat <unit>` | show the unit + its drop-ins concatenated |
| `sudo systemctl edit <unit>` | create/edit a drop-in override file (triggers its own daemon-reload) |
| `sudo systemctl mask <unit>` / `unmask` | make even manual `start` impossible / undo that |
| `systemctl is-enabled/is-active/is-failed <unit>` | script-friendly single word + exit code |
| `systemctl list-unit-files` | enable state of every unit |
| `systemctl list-dependencies <unit>` | dependency tree |

## 2.1 — PID 1: systemd

The kernel's first process at boot is systemd; every other process on the system (services, your shell, your SSH connection) is its child, directly or indirectly.

```
$ ps -p 1 -o pid,comm
    PID COMMAND
      1 systemd
```

| Flag | Meaning |
|---|---|
| `-p` | filter by PID |
| `-o pid,comm` | show only PID and process name (command) |

Practical relevance: `systemctl` commands work because you're talking to systemd. `kill -9 1` does NOT crash the machine — the kernel never delivers signals without an installed handler (including SIGKILL) to PID 1; they're silently ignored (source: `man 2 kill`, the "init special case"; `man 7 signal`).

## 2.2 — systemd is a daemon, systemctl is a client

`systemd` (PID 1) is a daemon that runs continuously in the background — starting/stopping services, reading unit files is its actual job. `systemctl` doesn't do the work itself; it sends systemd a "do this" message.

That communication happens over **D-Bus** — a Linux IPC (inter-process communication) system used for processes to talk to each other. `systemctl` sends a D-Bus message; systemd (the listener on D-Bus) receives and processes it. Not just systemd — many Linux services (NetworkManager, the bluetooth stack) use D-Bus too.

## 2.3 — Process tree: parent/child

`ps -ef --forest` draws the output as a tree — indentation and `└─` characters show which process is whose child. systemd (PID 1) sits at the top-left, everything else branches down from there.

| Command | For |
|---|---|
| `ps -ef --forest \| grep sshd` | filter to just the sshd branch |
| `ps -ef --forest \| less` then `/sshd` | interactive search; `n` next match, `q` quit |

```
systemd (PID 1)
 └─ sshd listener (857, root)               — always-running sshd, waiting for connections
     ├─ sshd: ubuntu [priv] (858)            — privilege-separation process for pts/0
     │   └─ sshd: ubuntu@pts/0 (1003)        — first SSH session
     │       └─ -bash (1049)
     └─ sshd: ubuntu [priv] (860)            — pts/1 (the one in use now)
         └─ sshd: ubuntu@pts/1 (1048)
             └─ -bash (1058)
                 ├─ ps -ef --forest (1144)
                 └─ less (1145)
```

Verifying the chain one hop at a time with `ps -p <pid> -o pid,ppid`:

| PID | PPID |
|---|---|
| 1058 | 1048 |
| 1048 | 860 |
| 860 | 857 |
| 857 | 1 |
| 1 | 0 |

PID 1's PPID is 0 — the kernel itself.

## 2.4 — systemd unit directories

| Directory | Priority | Written by |
|---|---|---|
| `/etc/systemd/system/` | high | sysadmin (hand-added/overridden units, symlinks created by `enable`) |
| `/run/systemd/system/` | mid | systemd/programs (runtime-generated temporary units, deleted on reboot) |
| `/usr/lib/systemd/system/` (symlink: `/lib/systemd/system/`) | low | apt/dpkg (default unit files installed by packages) |

| Config/state files | Holds |
|---|---|
| `/etc/systemd/*.conf` (`system.conf`, `journald.conf`, `logind.conf`...) | systemd's own behavior settings |
| `/var/lib/systemd/` | systemd's state files |
| `/var/log/journal/` | where journalctl logs live (if persistent) |

## 2.5 — Inspecting the directory: symlinks

```
$ ls -la /etc/systemd/system/ /lib/systemd/system/ 2>&1 | head -30
/etc/systemd/system/:
lrwxrwxrwx  1 root root   38 Jun 10 10:16 chronyd.service -> /usr/lib/systemd/system/chrony.service
drwxr-xr-x  2 root root 4096 Jun 10 10:10 cloud-config.target.wants
lrwxrwxrwx  1 root root   44 Jun 10 10:11 dbus-org.freedesktop.ModemManager1.service -> /usr/lib/systemd/system/ModemManager.service
lrwxrwxrwx  1 root root   48 Jun 10 10:08 dbus-org.freedesktop.resolve1.service -> /usr/lib/systemd/system/systemd-resolved.service
drwxr-xr-x  2 root root 4096 Sep 20 20:50 multi-user.target.wants
-rw-r--r--  1 root root  359 Jun 10 10:16 snap-amazon\x2dssm\x2dagent-13009.mount
-rw-r--r--  1 root root  591 Sep 20 20:50 snap.amazon-ssm-agent.amazon-ssm-agent.service
```

| Entry type | Example | What it means |
|---|---|---|
| symlink (starts with `l`) | `chronyd.service -> .../chrony.service` | someone ran `systemctl enable chronyd`; `enable` doesn't copy the file, it creates a symlink pointing to the original in `/usr/lib/systemd/system/` |
| `.wants`/`.requires` directory | `multi-user.target.wants/` | "when this target runs, these should run too" list — contents are always symlinks only |
| plain file (starts with `-`) | `snap-core22-2411.mount` | a real unit file, usually auto-generated (snap packages) |

Verification:

```
$ ls -l /etc/systemd/system/chronyd.service
lrwxrwxrwx 1 root root 38 Jun 10 10:16 /etc/systemd/system/chronyd.service -> /usr/lib/systemd/system/chrony.service
$ ls -l /usr/lib/systemd/system/chrony.service
-rw-r--r-- 1 root root 1923 Jul  2  2024 /usr/lib/systemd/system/chrony.service
```

## 2.6 — The `.wants` mechanism and targets

`multi-user.target` = the state where the system has "network up, multi-user capable, no graphical interface" — the normal boot target used on servers.

| Rule | Explanation |
|---|---|
| Contents of `.wants`/`.requires` | always symlinks only, never a real file |
| Which target it's linked to | depends on the `WantedBy=<target>` line in the unit file's `[Install]` section; `enable` drops the symlink into that target's `.wants/` directory |
| Relation to boot | at boot systemd tries to reach a default target (usually `multi-user.target`), and starts every service in that target's `.wants/` directory along the way |

## 2.7 — Unit file anatomy

```
$ cat /lib/systemd/system/cron.service
[Unit]
Description=Regular background program processing daemon
Documentation=man:cron(8)
After=remote-fs.target nss-user-lookup.target

[Service]
EnvironmentFile=-/etc/default/cron
ExecStart=/usr/sbin/cron -f -P $EXTRA_OPTS
IgnoreSIGPIPE=false
KillMode=process
Restart=on-failure
SyslogFacility=cron

[Install]
WantedBy=multi-user.target
```

| Section | Content |
|---|---|
| `[Unit]` | identity and dependencies — `Description=`, `After=`, `Before=`, `Requires=`, `Wants=` |
| `[Service]` | type-specific — `ExecStart=`, `ExecStop=`, `Restart=`, `Type=`, `User=` |
| `[Install]` | what happens on enable, which target to attach to — `WantedBy=`, `RequiredBy=`, `Alias=` |

| Unit type | Represents | Example |
|---|---|---|
| `.service` | a process/daemon | `cron.service`, `nginx.service` |
| `.mount` | a filesystem mount point | `boot-efi.mount` |
| `.socket` | a network/IPC socket — triggers the associated service on connection | `docker.socket` |
| `.timer` | a scheduled task, similar to cron | `apt-daily.timer` |
| `.target` | a synchronization point representing a group of units (not a real process) | `multi-user.target` |

### Case study: writing the `logger-demo` service

| Step | Command |
|---|---|
| write the script | `sudo tee /usr/local/bin/logger-demo.sh > /dev/null << 'EOF' ... EOF` + `sudo chmod +x` |
| write the unit file | `sudo tee /etc/systemd/system/logger-demo.service > /dev/null << 'EOF' ... EOF` |
| check it's loaded | `systemctl status logger-demo` |
| start | `sudo systemctl start logger-demo` |
| check status | `systemctl status logger-demo` |
| start at boot | `sudo systemctl enable logger-demo` |
| watch logs | `journalctl -u logger-demo -n 5` / `-f` |
| stop + disable | `sudo systemctl stop logger-demo` / `sudo systemctl disable logger-demo` |
| reboot test (disabled) | `sudo reboot` → `systemctl status logger-demo` |
| reboot test (enabled) | `sudo systemctl enable logger-demo` → `sudo reboot` → `systemctl status logger-demo` |

Script:
```bash
#!/bin/bash
counter=0
while true; do
  echo "$(date '+%Y-%m-%d %H:%M:%S') - heartbeat #$counter"
  counter=$((counter + 1))
  sleep 5
done
```

Unit:
```ini
[Unit]
Description=Demo heartbeat logger

[Service]
ExecStart=/usr/local/bin/logger-demo.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Process:

1. The moment the unit file is written to `/etc/systemd/system/`, `systemctl status logger-demo` already recognizes it — but since it hasn't been started, it's `inactive`:
   ```
   ○ logger-demo.service - Demo heartbeat logger
        Loaded: loaded (/etc/systemd/system/logger-demo.service; disabled; preset: enabled)
        Active: inactive (dead)
   ```
2. After `sudo systemctl start logger-demo` it's running, but `Loaded` still says `disabled` — **start and enable are independent steps**, starting a service doesn't mean it'll start at boot:
   ```
   ● logger-demo.service - Demo heartbeat logger
        Loaded: loaded (/etc/systemd/system/logger-demo.service; disabled; preset: enabled)
        Active: active (running) since Mon 2026-09-21 22:00:05 UTC; 9s ago
      Main PID: 1901 (logger-demo.sh)
         Tasks: 2 (limit: 265)
        Memory: 856.0K (peak: 1.1M)
           CPU: 12ms
        CGroup: /system.slice/logger-demo.service
                ├─1901 /bin/bash /usr/local/bin/logger-demo.sh
                └─1907 sleep 5
   ```
   `ls /etc/systemd/system/multi-user.target.wants/ | grep logger` returns **empty** at this point — it hasn't been `enable`d yet.
3. `sudo systemctl enable logger-demo` does two things: `Loaded` now says `enabled`, and a symlink appears in `.wants/`:
   ```
   $ sudo systemctl enable logger-demo
   Created symlink /etc/systemd/system/multi-user.target.wants/logger-demo.service → /etc/systemd/system/logger-demo.service.
   $ ls -la /etc/systemd/system/multi-user.target.wants/logger-demo.service
   lrwxrwxrwx 1 root root 39 Sep 21 22:03 ... -> /etc/systemd/system/logger-demo.service
   ```
4. `journalctl -u logger-demo -n 5` shows the service's logs; `-f` follows live (`^C` to exit):
   ```
   Sep 21 22:05:05 lev-k logger-demo.sh[1901]: 2026-09-21 22:05:05 - heartbeat #60
   Sep 21 22:05:10 lev-k logger-demo.sh[1901]: 2026-09-21 22:05:10 - heartbeat #61
   ```
5. `stop` kills the process with `SIGTERM` (`code=killed, signal=TERM`), `disable` removes the symlink:
   ```
   $ sudo systemctl disable logger-demo
   Removed "/etc/systemd/system/multi-user.target.wants/logger-demo.service".
   ```
6. **Reboot test 1 (disabled):** before `sudo reboot`, `Loaded: disabled` / `Active: inactive`. Same after reboot: still `disabled` / `inactive` — a disabled service doesn't come up on its own at reboot.
7. **Reboot test 2 (enabled):** after `sudo systemctl enable logger-demo` the `.wants/` symlink reappears, `Loaded: enabled` but still `Active: inactive` (not started yet). After `sudo reboot`, the service is running again automatically **with a new PID**:
   ```
   ● logger-demo.service - Demo heartbeat logger
        Loaded: loaded (/etc/systemd/system/logger-demo.service; enabled; preset: enabled)
        Active: active (running) since Mon 2026-09-21 22:15:36 UTC; 24s ago
      Main PID: 525 (logger-demo.sh)
   Sep 21 22:15:36 lev-k systemd[1]: Started logger-demo.service - Demo heartbeat logger.
   ```
   Process state is never preserved across reboot (the old PID 1901 is gone, new PID 525); the only thing that survives is the symlink in `.wants/` — i.e. the "this service should start at boot" fact.
8. **Round 2 (a few days later): `logger-demo` was deleted and recreated with the same script + unit file content, to live-test `Restart=`/`StartLimitBurst` behavior.** First, the current settings:
   ```
   $ systemctl show logger-demo -p Restart,RestartUSec,StartLimitBurst,StartLimitIntervalUSec
   Restart=on-failure
   RestartUSec=100ms          (default, we didn't set it)
   StartLimitIntervalUSec=10s (default)
   StartLimitBurst=5          (default)
   ```
   | `Restart=` value | When it restarts |
   |---|---|
   | `no` (default) | never restarts automatically |
   | `on-failure` | the process exits with an error (non-zero exit, or is killed by a signal other than SIGTERM/SIGINT/SIGHUP/SIGPIPE) |
   | `on-abnormal` | terminated by a signal, times out, or the watchdog fires (a clean exit/normal stop doesn't count) |
   | `always` | restarts no matter how it exits (including a clean stop) |

   `StartLimitBurst=5` / `StartLimitIntervalUSec=10s` = the rate limit "stop if there are more than 5 restart attempts within 10 seconds".
9. **Debugging a broken service:** a deliberately non-existent `EnvironmentFile` was added to the unit file:
   ```
   $ sudo sed -i '/ExecStart=/i EnvironmentFile=/etc/logger-demo-env-yok' /etc/systemd/system/logger-demo.service
   $ sudo systemctl daemon-reload
   $ sudo systemctl restart logger-demo
   Job for logger-demo.service failed...
   $ systemctl status logger-demo
   Active: activating (auto-restart) (Result: resources)
   ```
   A few seconds later the rate limit kicks in:
   ```
   × logger-demo.service - Demo heartbeat logger v2
        Active: failed (Result: resources) since ...; 12s ago
   Sep 26 15:22:30 lev-k systemd[1]: logger-demo.service: Scheduled restart job, restart counter is at 5.
   Sep 26 15:22:30 lev-k systemd[1]: logger-demo.service: Start request repeated too quickly.
   Sep 26 15:22:30 lev-k systemd[1]: logger-demo.service: Failed with result 'resources'.
   ```
   | Symbol | Meaning |
   |---|---|
   | `●` | healthy/active |
   | `○` | inactive |
   | `×` | failed |

   The `resources` result category means the failure came from systemd's process-launching machinery, not the process's own exit code (here: the `EnvironmentFile` couldn't be read). Other categories: `exit-code`, `signal`, `timeout`, `core-dump`.
   ```
   $ systemctl is-failed logger-demo
   failed
   $ sudo systemctl reset-failed logger-demo
   $ systemctl is-failed logger-demo
   inactive
   ```
   Without `reset-failed`, even a restart attempt would be rejected (the rate limit is still active). After fixing the root cause (the `-` prefix, see 2.11), it was restarted again:
   ```
   $ sudo sed -i 's#EnvironmentFile=/etc/logger-demo-env-yok#EnvironmentFile=-/etc/logger-demo-env-yok#' /etc/systemd/system/logger-demo.service
   $ sudo systemctl daemon-reload
   $ sudo systemctl restart logger-demo
   $ systemctl is-failed logger-demo
   active
   ```
   Log evidence (`-p err` = only error+ severity):
   ```
   $ journalctl -u logger-demo -p err
   Sep 26 15:22:30 lev-k systemd[1]: logger-demo.service: Failed to load environment files: No such file or directory
   Sep 26 15:22:30 lev-k systemd[1]: Failed to start logger-demo.service - Demo heartbeat logger v2.
   ```
   This pair of lines repeats 5 times — matching `StartLimitBurst=5` exactly.

## 2.8 — `daemon-reload`: when it's needed

systemd notices a brand-new unit file automatically via `inotify` (no warning); it does not notice changes to an existing file, and prints a warning instead. `daemon-reload` rescans ALL unit files (active and inactive), and never stops/starts any process — it only refreshes systemd's unit metadata.

```
$ sudo sed -i 's/Description=Demo heartbeat logger/Description=Demo heartbeat logger v2/' /etc/systemd/system/logger-demo.service
$ systemctl status logger-demo
Warning: The unit file, source configuration file or drop-ins of logger-demo.service changed on disk. Run 'systemctl daemon-reload' to reload units.
● logger-demo.service - Demo heartbeat logger
     Active: active (running) since Sat 2026-09-26 15:04:46 UTC; 1min 32s ago
   Main PID: 2543 (logger-demo.sh)

$ sudo systemctl daemon-reload
$ systemctl status logger-demo
● logger-demo.service - Demo heartbeat logger v2
     Active: active (running) since Sat 2026-09-26 15:04:46 UTC; 3min 52s ago
   Main PID: 2543 (logger-demo.sh)
```

| Situation | What happens |
|---|---|
| a new unit file is created | loaded automatically (inotify), no warning |
| an existing unit file is edited | not loaded automatically, "changed on disk" warning appears |
| `daemon-reload` | rescans all unit files; Main PID/since **don't change** — even if `ExecStart=` changed, the process keeps running with the old command until it's also `restart`ed |

## 2.9 — `Type=` semantics

Even if `Type=` is never written in `[Service]`, systemd always computes a value for it; the default is `simple` (`man 5 systemd.service`).

```
$ systemctl show logger-demo -p Type
Type=simple
$ systemctl show ssh -p Type
Type=notify
```

`ssh` has `Type=notify` written by hand because it genuinely supports the `sd_notify` protocol (see `systemctl cat ssh`, 2.13).

| `Type=` | Behavior |
|---|---|
| `simple` (default) | considered started as soon as `ExecStart=` is forked |
| `exec` | like `simple`, but waits until the process is actually `exec()`'d |
| `forking` | considered started once the main process forks itself into the background (`PIDFile=` reports the real PID) |
| `oneshot` | considered started once the process exits; `RemainAfterExit=yes` keeps it showing "active" |
| `notify` | the process itself tells systemd it's ready via a `READY=1` message |

## 2.10 — `Wants=`/`Requires=` vs `After=`/`Before=`

`Wants=`/`Requires=` = "is this needed" (presence/absence, starts it automatically). `After=`/`Before=` = "in what order" (timing only, never starts anything). They're independent of each other — usually written together (`Wants=X` + `After=X`).

Trap: `network.target` does NOT mean "network is ready", only that "network infrastructure has been triggered". A real "network ready" guarantee needs `network-online.target` + `Wants=network-online.target` (a common gotcha on AWS/cloud-init boxes).

```
$ systemctl cat ssh
...
After=network.target auditd.service
```

(Only `After` is present, no `Wants`/`Requires` — ssh assumes `network.target` is already started from somewhere else, it doesn't start it itself.)

```
$ cat /usr/lib/systemd/system/basic.target
[Unit]
Description=Basic System
Documentation=man:systemd.special(7)
Requires=sysinit.target
Wants=sockets.target timers.target paths.target slices.target
After=sysinit.target sockets.target paths.target slices.target tmp.mount
RequiresMountsFor=/var /var/tmp
Wants=tmp.mount
```

`Requires=sysinit.target` + `After=sysinit.target` together — proof of the "pairing" pattern. `/var`, `/var/tmp` are MANDATORY (`RequiresMountsFor=`), `/tmp` is only SOFT (`Wants=tmp.mount`) because `tmp.mount` might be masked, and that shouldn't count as an error.

`logger-demo`'s own (implicit) dependencies:

```
$ systemctl show logger-demo -p After,Wants,Requires,WantedBy
Requires=sysinit.target system.slice
Wants=
WantedBy=
After=basic.target systemd-journald.socket sysinit.target system.slice
```

We wrote nothing in `[Unit]`, yet `Requires=`/`After=` are populated — `DefaultDependencies=yes` (the default, we never turned it off) is systemd automatically adding implicit dependencies (`systemctl show logger-demo -p DefaultDependencies` → `yes`).

`WantedBy=` comes back empty — even though `[Install]` says `WantedBy=multi-user.target` — because that only becomes a REAL link once the unit is `enable`d (the symlink exists); while disabled it's empty. `[Install]` is an "intent" written in the file, not an actual link.

## 2.11 — The `-` prefix: swallowing errors (`EnvironmentFile=-...`)

The `-` prefix means "ignore it if this file/command fails or doesn't exist". `EnvironmentFile=-/etc/x` lets the service start even if the file is missing — that was exactly the fix in step 9 of the case study. The same pattern is in the cron unit: `EnvironmentFile=-/etc/default/cron` (see 2.7).

```
$ systemctl show logger-demo -p EnvironmentFile
(empty — wrong property name, silently returns nothing, no error)
$ systemctl show logger-demo -p EnvironmentFiles
EnvironmentFiles=/etc/logger-demo-env-yok (ignore_errors=yes)
```

The correct/real property name is PLURAL: `EnvironmentFiles`. The `-` prefix turns into the `ignore_errors=yes` flag.

Full-loop verification — a real env file plus a drop-in `Environment=` were tested together:

```
$ echo 'REAL_VAR=merhaba' | sudo tee /etc/logger-demo-env-yok
$ journalctl -u logger-demo -n 3
... heartbeat #0 - DEMO_VAR=hello REAL_VAR=merhaba
```

Both (`Environment=` from the drop-in, `EnvironmentFile=` from the real file) were correctly injected into the process.

## 2.12 — cgroups: `systemd-cgls`, `stop` vs. a bare `kill`, SIGTERM vs. SIGKILL

systemd runs every service in its own cgroup; `stop` kills the ENTIRE cgroup, not just the Main PID — child processes (like `sleep`) aren't left orphaned.

```
$ systemd-cgls /system.slice/logger-demo.service
CGroup /system.slice/logger-demo.service:
├─3346 /bin/bash /usr/local/bin/logger-demo.sh
└─3379 sleep 5
```

Live test — killing the main PID with a bare `kill` (not `systemctl stop`):

```
$ sudo kill 3346        # SIGTERM, default
→ Active: inactive (dead), "Deactivated successfully" — RESTART NOT TRIGGERED
```

Why (`man 5 systemd.service`, the `Restart=on-failure` definition): "is terminated by a signal (excluding SIGHUP, SIGINT, SIGTERM, SIGPIPE)". `SIGTERM` is deliberately treated as a "clean shutdown", because systemd's own `stop` command also uses `SIGTERM` — a bare `kill` is treated the same as `systemctl stop`.

```
$ sudo kill -9 <PID>    # SIGKILL, not on the exclusion list
Sep 26 15:28:08 lev-k systemd[1]: logger-demo.service: Scheduled restart job, restart counter is at 1.
Sep 26 15:28:08 lev-k systemd[1]: Started logger-demo.service - Demo heartbeat logger v2.
→ RESTART TRIGGERED, new PID, heartbeat starting again from #0.
```

| Signal | Does `Restart=on-failure` trigger |
|---|---|
| `SIGTERM`, `SIGINT`, `SIGHUP`, `SIGPIPE` | no — counts as a clean shutdown |
| `SIGKILL` and every other signal | yes |

Note: same source (`man 7 signal`) as the `kill -9 1` correction in 2.1 — SIGKILL is the one signal that can never have a handler installed, and kills any process directly except PID 1.

## 2.13 — `systemctl cat`: viewing a unit + its drop-ins together

Prints the unit plus all of its drop-ins concatenated into their combined/effective form, with the source file paths shown as comments (it's just concatenation — it doesn't make the override decision itself).

```
$ systemctl cat ssh
# /usr/lib/systemd/system/ssh.service
[Unit]
Description=OpenBSD Secure Shell server
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target auditd.service
ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

[Service]
EnvironmentFile=-/etc/default/ssh
ExecStartPre=/usr/sbin/sshd -t
ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
ExecReload=/usr/sbin/sshd -t
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartPreventExitStatus=255
Type=notify
RuntimeDirectory=sshd
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
Alias=sshd.service

# /usr/lib/systemd/system/ssh.service.d/ec2-instance-connect.conf
[Service]
ExecStart=
ExecStart=/usr/sbin/sshd -D -o "AuthorizedKeysCommand /usr/share/ec2-instance-connect/eic_run_authorized_keys %%u %%f" -o "AuthorizedKeysCommandUser ec2-instance-connect" $SSHD_OPTS
```

Important detail: `ExecStart=` appears TWICE in the drop-in — first EMPTY (= "reset everything accumulated so far"), then the new value. Some directives like `ExecStart=` are normally "cumulative" — without the empty line, both would try to run stacked on top of each other. This is the official technique AWS's EC2 Instance Connect package uses to COMPLETELY REPLACE the original `sshd` command.

| | `logger-demo.service` | `ssh.service` |
|---|---|---|
| Main file | `/etc/systemd/system/` — we wrote it by hand | `/usr/lib/systemd/system/` — installed by a package |
| Override/drop-in | `/etc/systemd/system/logger-demo.service.d/` (see 2.14) | `/usr/lib/systemd/system/ssh.service.d/` (installed by the `ec2-instance-connect` package) |

General rule: `/usr/lib/` = the vendor/package layer (`apt upgrade` can overwrite it), `/etc/` = the admin layer (apt never touches it, your overrides always live here). If a unit of the same name exists in both, `/etc/` wins.

## 2.14 — Drop-in files and `systemctl edit`

The official way to override a unit's behavior without editing the unit file itself.

```
$ sudo systemctl edit logger-demo
→ opens an editor, Environment=DEMO_VAR=hello was added under [Service], saved.
"Successfully installed edited file '/etc/systemd/system/logger-demo.service.d/override.conf'."
```

(It triggers its own `daemon-reload` — no need to do it by hand, unlike `sudo tee`.)

```
$ cat /etc/systemd/system/logger-demo.service.d/override.conf
[Service]
Environment=DEMO_VAR=hello
$ systemctl show logger-demo -p Environment
Environment=DEMO_VAR=hello
```

Verified in full during the 2.11 full-loop test — `journalctl` showed both `Environment=` and `EnvironmentFile=` were genuinely injected into the process.

## 2.15 — `mask`/`unmask`

`mask` is stronger than `disable`; it drops a `/dev/null` symlink at `/etc/systemd/system/<unit>` — even a manual `start` becomes impossible. `unmask` removes the symlink and restores the previous state.

Trap: `logger-demo`'s REAL file already lives in `/etc/`, so masking it directly failed:

```
$ sudo systemctl mask logger-demo
Failed to mask unit: File /etc/systemd/system/logger-demo.service already exists.
```

(`logger-demo` was left untouched — `--force` doesn't change this either: per `man systemctl`, `--force` is mainly for resolving symlink conflicts in `enable`/`link`/`unmask`, it doesn't overwrite a real file for `mask`.)

Tested with a separate dummy (`mask-test.service`), moved into `/usr/lib/systemd/system/` (simulating a package install) for a realistic scenario:

```
$ sudo systemctl mask mask-test
Created symlink /etc/systemd/system/mask-test.service → /dev/null.
$ sudo systemctl start mask-test
Failed to start mask-test.service: Unit mask-test.service is masked.
```

(With `disable`, a manual `start` would still work — with `mask` it doesn't; that's the real difference.)

```
$ sudo systemctl unmask mask-test
Removed "/etc/systemd/system/mask-test.service".
$ sudo systemctl start mask-test
$ systemctl status mask-test
○ mask-test.service - Mask test dummy
     Loaded: loaded (/usr/lib/systemd/system/mask-test.service; static)
     Active: inactive (dead)
```

(The original in `/usr/lib/` was found automatically — it ran successfully, content was never lost.)

Note: `static` enablement state — units with no `[Install]` section show up this way; they can't be `enable`d/`disable`d, only started manually or via a dependency.

## 2.16 — `is-enabled` / `is-active` / `is-failed`

```
$ systemctl is-enabled logger-demo
disabled
$ systemctl is-active logger-demo
active
$ systemctl is-failed logger-demo
active
```

Watch out: `is-failed` actually just prints `ActiveState` too — there's no separate word for "not failed", only the EXIT CODE differs (0 = actually failed, non-zero otherwise). In scripts, check `$?`, not the printed word. All three are automation/script-friendly: a single word plus a meaningful exit code.

## 2.17 — `list-unit-files` / `list-dependencies` / `list-units`

```
$ systemctl list-unit-files | grep logger-demo
logger-demo.service                            disabled        enabled
```

Two columns: STATE (the real, current state — this drives boot behavior) vs. VENDOR PRESET (a SUGGESTION coming from `/usr/lib/systemd/system-preset/*.preset` files, with no enforcing effect at all — only applied via the `systemctl preset` command).

```
$ systemctl list-dependencies logger-demo
logger-demo.service
● ├─system.slice
● └─sysinit.target
●   ├─apparmor.service
●   ├─blk-availability.service
    ...
●   ├─local-fs.target
●   │ ├─-.mount
●   │ ├─boot-efi.mount
```

Symbols: `●` = active/running, `○` = inactive (some units are `oneshot`, so having run once at boot and stopped is normal). `list-dependencies` is recursive by default — targets also expand their own dependencies (like mounts under `local-fs.target`).

## Relationship to Kubernetes

- On a Kubernetes node, `kubelet` is just an ordinary systemd-managed service on the host; `systemctl status kubelet` / `journalctl -u kubelet -f` work exactly the way we learned here.
- The container runtime (`containerd` or `CRI-O`) also runs as its own systemd unit — kubelet talks to it over CRI (Container Runtime Interface), and systemd keeps both of them alive independently.
- **cgroup driver**: both kubelet and the runtime manage container resource limits (cgroups) using either the `systemd` driver or the older `cgroupfs` driver. If they use different drivers, the machine ends up with two separate cgroup management authorities, and the node can become unstable.
- That's why kubelet and the runtime must use the **same** cgroup driver; on systemd-based distros (Ubuntu included) the recommended one is the `systemd` driver, since systemd is already the single cgroup manager on the box.
- The same model holds on OpenShift/RHEL CoreOS: kubelet and CRI-O also run as systemd units, and node configuration (the machine-config-operator) updates those same unit files.

## Notes

| Flag | Meaning |
|---|---|
| `ps -p` | filter by PID |
| `ps -o pid,comm` / `pid,ppid` | show only the requested columns |
| `ps -ef --forest` | tree view |

- `start` and `enable` are independent: `start` runs it now, `enable` makes it run at boot. Doing one doesn't do the other.
- The contents of `.wants`/`.requires` directories are always symlinks; the unit file itself lives in `/etc/systemd/system/` or `/usr/lib/systemd/system/`.
- The reboot test showed process state isn't preserved (new PID); the only thing that persists is the enabled state (whether the symlink exists).
- `daemon-reload` only refreshes metadata, it never restarts a process — an `ExecStart=` change needs an explicit `restart` on top to take effect.
- `systemctl show -p <property>` asks for the REAL property name exposed over D-Bus, which may not match the directive name in the unit file 1:1 (e.g. the `EnvironmentFile=` directive maps to the `EnvironmentFiles` property, plural). A wrong/unknown property name silently returns nothing, no error.
- `mask` fails if a real file already exists at `/etc/systemd/system/` (`--force` doesn't change this); the real difference between `disable` and `mask` is whether a manual `start` still works.

[↑ Go back to TOC](#table-of-contents)

---

# Part 3 — Logging & journalctl

Machine: Ubuntu 24.04 (AWS), account `ubuntu` (in the `adm` group — that's where read access to journalctl/`/var/log` comes from).

## Cheat sheet

| Command | What it does |
|---|---|
| `logger "msg"` | send a test message (classic `syslog()` call, via `/dev/log`) |
| `logger -p crit "msg"` | send with a given priority (`crit`+ messages trigger an immediate fsync in journald) |
| `logger -u <socket> "msg"` | write directly to a Unix socket instead of `/dev/log` |
| `journalctl -u <unit>` | show logs for a given unit |
| `journalctl -u <unit> -n N` | last N lines |
| `journalctl --since ... --until ...` | filter by time range |
| `journalctl -p <sev>` / `-p a..b` | filter by severity (single value or range, both ends inclusive) |
| `journalctl -f` | follow live |
| `journalctl -b [-N]` | logs for a specific boot (`-1` = the previous boot) |
| `journalctl --file=<path>` | read a specific journal file directly |
| `journalctl --flush` | move logs from RAM (`/run/log/journal`) to disk (`/var/log/journal`) |
| `journalctl --relinquish-var` | let go of the disk, new writes go to RAM |
| `journalctl --sync` | force fsync on all open journal files right now |
| `systemd-analyze cat-config systemd/journald.conf` | show the merged/effective config across all layers |
| `systemctl is-active rsyslog` | is rsyslog running |
| `strace -f -e trace=fsync,fdatasync -p $(pidof systemd-journald)` | watch live when journald actually calls fsync |

## 3.1 — Architecture: entry points → journald → file (+ syslog mirror)

### Diagram 1 — basic flow: sources → journald → storage

![journald architecture - level 1](misc/journald_architecture_mermaid.png)

The simplest picture: the entry points (kernel ring buffer, syslog(), native API, systemd stdout/stderr, kernel audit) are collected by journald, written to persistent (disk) or volatile (RAM) storage, and forwarded to syslog/kmsg/console/wall if configured. No mechanism detail yet, just the flow.

### Diagram 2 — + rate limiting, mmap, forward destination detail

![journald architecture - level 2](misc/journald_architecture_mermaid_2.png)

Added on top of the previous diagram: journald applying filters/rate limiting to messages, accessing the journal file via `mmap`, and the forward destination spelled out — rsyslog is now its own box writing to its actual files (`/var/log/syslog`, `/var/log/auth.log`, `/var/log/kern.log`).

### Diagram 3 — comprehensive: config layers, durability, rotation, the RAM distinction

![journald architecture - level 3](misc/journald_architecture_mermaid_3.png)

The most complete picture. Added on top: the config file layers (`journald.conf` + drop-ins), all four `Storage=` values, the return to RAM at shutdown via `--smart-relinquish-var`, the durability chain (kernel writeback → `SyncIntervalSec` → immediate sync on CRIT+ messages), rotation/retention settings, and the most critical point: the volatile journal in RAM (tmpfs, `/run/log/journal`) is NOT the same thing as the persistent file's page cache in RAM.

| `Storage=` value | What happens |
|---|---|
| `persistent` | created if disk isn't there, always written to disk |
| `auto` (default) | disk if `/var/log/journal` exists, RAM otherwise |
| `volatile` | always RAM (tmpfs), lost on reboot |
| `none` | nothing written to a journal file at all, only forwarding works |

| Critical distinction | Meaning |
|---|---|
| `--flush` vs `--sync` | flush: moves the volatile journal to the persistent journal; sync: wait for already-written records to hit disk (fsync) |
| `SyncIntervalSec` | doesn't mean "every record sits in RAM for this long" — kernel writeback can flush it earlier |
| append-based ≠ append-only | records are written by appending, but the file header and indexes can still be updated |

A message can enter journald through 4 different gates: the kernel ring buffer (`/dev/kmsg`), the classic `syslog()` call (`/dev/log`), the native `sd_journal_send()` socket, and units' stdout/stderr. All of it is collected by journald, trusted fields are added (`_PID`, `_UID`, `_COMM`, `_SYSTEMD_UNIT`... — a client can't forge these), and it's written to journald's own binary `*.journal` file. If `ForwardToSyslog=yes` is set, a copy of the same message also goes to rsyslog.

```
logger "msg"
   │ (syslog() call, via /dev/log)
   ▼
systemd-journald ──────────► *.journal (binary)
   │
   │ (ForwardToSyslog=yes)
   ▼
/run/systemd/journal/syslog (socket)
   │
   ▼
rsyslog ───────────────────► /var/log/syslog (text)
```

Test: does a single `logger` command show up in two different systems, in two different formats (binary vs text)?

```
$ logger "Naber journalctl!"
$ journalctl -n 3
Sep 24 20:38:20 lev-k ubuntu[1202]: Naber journalctl!
$ systemctl is-active rsyslog
active
$ sudo tail -3 /var/log/syslog
2026-09-24T20:40:08.081218+00:00 lev-k ubuntu: Naber journalctl!
```

The same message showed up on both sides. Source drop-in:

```
$ cat /usr/lib/systemd/journald.conf.d/syslog.conf
[Journal]
ForwardToSyslog=yes
```

To verify the direction, we tried skipping journald entirely and writing straight to the socket rsyslog listens on:

```
$ echo "<13>socat test mesaji" | socat - UNIX-SENDTO:/run/systemd/journal/syslog
$ logger -u /run/systemd/journal/syslog "logger ile direkt socket testi"
$ tail -f /var/log/syslog
2026-09-24T20:56:12.871346+00:00 lev-k socat test mesaji
2026-09-24T20:57:41.352612+00:00 lev-k ubuntu: logger ile direkt socket testi
$ journalctl | grep -E "socat test|direkt socket testi"
                                            # (empty — no match at all)
```

| Method | Shows in journalctl | Shows in syslog | Why |
|---|---|---|---|
| `logger` (normal, `/dev/log`) | yes | yes (via forwarding) | goes through journald first |
| writing directly to the socket (`socat`/`logger -u`) | no | yes | skips journald entirely, lands straight on the socket rsyslog listens to |

This is the first proof that journald and rsyslog are two independent systems — one doesn't quietly run behind the other, there's only a one-way copy relationship between them.

## 3.2 — File inventory and config layers

Where journald's files/directories actually live (disk vs RAM), and where you need to write when you want to change a setting.

| Path | What | Persistent |
|---|---|---|
| `/usr/lib/systemd/systemd-journald` | daemon binary | disk |
| `/usr/bin/journalctl` | client — reads files directly, doesn't ask the daemon | disk |
| `/etc/systemd/journald.conf` | admin main config | disk |
| `/etc/systemd/journald.conf.d/*.conf` | admin drop-in | disk |
| `/run/systemd/journald.conf.d/*.conf` | runtime drop-in | RAM |
| `/usr/lib/systemd/journald.conf.d/*.conf` (`syslog.conf`) | vendor drop-in | disk |
| `/var/log/journal/<machine-id>/` | persistent log data | disk |
| `/run/log/journal/<machine-id>/` | volatile log data | RAM (tmpfs) |
| `/run/systemd/journal/{dev-log,socket,stdout,syslog}` | AF_UNIX sockets | RAM |

Config layer priority: the main file is read first, then all drop-ins in alphabetical order; if the same drop-in name exists in multiple layers, **`/etc` beats `/run` beats `/usr/lib`**.

```
$ systemd-analyze cat-config systemd/journald.conf
# /etc/systemd/journald.conf
...
[Journal]
ForwardToSyslog=yes
```

`ForwardToSyslog`'s compile-time default is `no` (the `#ForwardToSyslog=no` line in the main file is a comment, has no effect); the effective `yes` comes from the vendor drop-in (`/usr/lib/.../syslog.conf`) — Ubuntu's deliberate choice for compatibility with the old syslog pipeline.

We wrote our own drop-in and verified it:

```
$ printf '[Journal]\nCompress=no\n' | sudo tee /etc/systemd/journald.conf.d/99-lab.conf
$ sudo systemctl restart systemd-journald
$ systemd-analyze cat-config systemd/journald.conf | tail -n 10
# /etc/systemd/journald.conf.d/99-lab.conf
[Journal]
Compress=no

# /usr/lib/systemd/journald.conf.d/syslog.conf
[Journal]
ForwardToSyslog=yes
```

Both drop-ins side by side, both effective — as long as keys don't collide, layers merge; the priority rule only kicks in when the same key is defined in more than one place.

Cleanup: `sudo rm /etc/systemd/journald.conf.d/99-lab.conf && sudo systemctl restart systemd-journald && sudo rmdir /etc/systemd/journald.conf.d/`

## 3.3 — Storage= and the flush flow (RAM ↔ disk)

`Storage=` decides where journald writes: `auto` (Ubuntu's default) = `/var/log/journal` if it exists (behaves like persistent, doesn't create the directory itself); `volatile` = `/run/log/journal` only (RAM), gone on reboot; `none` = don't write at all, just forward.

| Directory | Storage | Mount source |
|---|---|---|
| `/var/log/journal/<machine-id>/` | Disk | `/dev/root` |
| `/run/log/journal/<machine-id>/` | RAM | `tmpfs` |

```
$ df -h /var /run
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        19G  2.8G   16G  15% /
tmpfs            83M  2.0M   82M   3% /run
```

Test: write a `Storage=volatile` drop-in and restart, compare before/after reboot:

```
$ printf '[Journal]\nStorage=volatile\n' | sudo tee /etc/systemd/journald.conf.d/99-lab.conf
$ sudo systemctl restart systemd-journald
$ logger "volatile reboot testi"
$ journalctl | grep "volatile reboot testi"
Sep 25 14:06:37 lev-k ubuntu[7508]: volatile reboot testi
$ grep "volatile reboot testi" /var/log/syslog
2026-09-25T14:06:37.908970+00:00 lev-k ubuntu: volatile reboot testi

$ sudo reboot
```

After reboot:

```
$ journalctl | grep "volatile reboot testi"
                                            # (empty)
$ grep "volatile reboot testi" /var/log/syslog
2026-09-25T14:06:37.908970+00:00 lev-k ubuntu: volatile reboot testi
```

With `Storage=volatile`, the journal lives in RAM (`/run/log/journal`) and resets on reboot — the message is gone from journalctl. `/var/log/syslog` was unaffected, because rsyslog writes to its own independent text file and has no idea what journald's Storage setting is.

`--flush` and `--relinquish-var` switch between these two modes **live, without a restart**:

| Flag | Direction | What it does |
|---|---|---|
| `--flush` | RAM → Disk | moves accumulated logs from `/run/log/journal` to `/var/log/journal`, subsequent writes go to disk |
| `--relinquish-var` | Disk → RAM | lets go of the disk, subsequent writes go to RAM (old records on disk aren't deleted) |

Live switching test (first reverted to a clean `persistent` state: `sudo rm /etc/systemd/journald.conf.d/99-lab.conf && sudo systemctl restart systemd-journald`):

**Test A — before/after `--relinquish-var`:**

```
$ logger "relinquish test A"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "relinquish test A"
Failed to open files: No such file or directory        # not in RAM — directory doesn't exist yet
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "relinquish test A"
Sep 25 21:31:32 lev-k ubuntu[13223]: relinquish test A  # on disk
$ grep "relinquish test A" /var/log/syslog
2026-09-25T21:31:32.261966+00:00 lev-k ubuntu: relinquish test A

$ sudo journalctl --relinquish-var
$ logger "relinquish test B"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "relinquish test B"
Sep 25 21:32:12 lev-k ubuntu[13237]: relinquish test B  # now in RAM
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "relinquish test B"
                                            # (empty — disk no longer growing)
$ grep "relinquish test B" /var/log/syslog
2026-09-25T21:32:12.206210+00:00 lev-k ubuntu: relinquish test B
```

**Test B — before/after `--flush`:**

```
$ logger "flush test A"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "flush test A"
Sep 25 21:32:45 lev-k ubuntu[13245]: flush test A       # still in RAM
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "flush test A"
                                            # (empty)

$ sudo journalctl --flush
$ logger "flush test B"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "flush test B"
Failed to open files: No such file or directory        # directory is gone now
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "flush test B"
Sep 25 21:35:38 lev-k ubuntu[13278]: flush test B       # back on disk
```

`/var/log/syslog` never changed across either test — rsyslog is fully independent of Storage transitions.

## 3.4 — Durability: fsync timing and the crash scenario

journald writes a message via `mmap` into page cache (RAM) first; it doesn't land on disk (`fsync`) immediately. We watched the journald process with `strace` to see which messages hold off fsync and which trigger it instantly.

```
# terminal 1
$ sudo strace -f -e trace=fsync,fdatasync -p $(pidof systemd-journald)
strace: Process 13215 attached

# terminal 2
$ logger "strace normal mesaj"
```

Nothing showed up in terminal 1.

```
# terminal 2
$ logger -p crit "strace crit mesaj"
```
```
# terminal 1
fsync(18)                               = 0
```

```
# terminal 2
$ sudo journalctl --sync
```
```
# terminal 1
[pid 13319] fsync(18 ...)               = 0
[pid 13215] fsync(19 ...)               = 0
```

| Sent | Seen in terminal 1 | Meaning |
|---|---|---|
| `logger "normal msg"` | nothing | normal priority holds off fsync, deferred to the `SyncIntervalSec` timer (default 5min) |
| `logger -p crit "..."` | instant `fsync` | `crit`/`alert`/`emerg` messages don't wait for the timer, trigger sync instantly |
| `sudo journalctl --sync` | instant `fsync` (multiple files) | manual force, flushes all open journal files to disk right away |

Crash test: we secured one message with `--sync`, left one message unsecured, and instantly reset the machine (no shutdown procedure) with `sysrq-trigger`:

```
$ logger "crash test SYNCED"
$ sudo journalctl --sync
$ logger "crash test UNSYNCED" && echo b | sudo tee /proc/sysrq-trigger
                                            # connection dropped instantly here
```

After reboot:

```
$ journalctl -b -1 | grep "crash test"
Sep 25 21:52:44 lev-k ubuntu[13329]: crash test SYNCED
$ grep "crash test" /var/log/syslog
                                            # (empty — NEITHER shows up, including SYNCED)
```

| Message | On journald's side (`-b -1`) | On syslog's side |
|---|---|---|
| `crash test SYNCED` (+ `--sync`) | survived | **lost** |
| `crash test UNSYNCED` | lost | lost |

Unexpected but important result: `--sync` only guarantees journald's own binary file, it never forces rsyslog to sync its own text file. rsyslog has its own independent buffer/sync policy; it hadn't hit disk yet at the moment of the crash, so it was lost too. Even a message that's "guaranteed" on journald's side can remain unguaranteed on syslog's side — concrete proof that the two systems are genuinely independent.

## 3.5 — journalctl reference (remaining commands)

Beyond what was tested live above, these are additional filtering/format/cleanup flags mentioned in the notes but not covered as a case study. Pure reference table — some (`--disk-usage`, `--vacuum-*`, `--rotate`) were tried on the machine, the rest weren't.

| Command | What it does |
|---|---|
| `journalctl --disk-usage` | shows the journal's total disk usage |
| `sudo journalctl --vacuum-size=SIZE` | deletes archived files down to a size limit — **never touches the active file**, so total usage may stay above the limit. The trailing `~` on deleted archive names marks an "unclean shutdown" archive |
| `sudo journalctl --rotate` | rotates the active file into an archive and opens a new active file — runs silently, effect is confirmed via `journalctl -u systemd-journald` |
| `journalctl -S <time>` / `-U <time>` | short form of `--since`/`--until` |
| `journalctl -b` / `-b -1` | logs for a specific boot (`-1` = the previous boot) — `-b -1` was used live in the crash test (3.4) |
| `journalctl --list-boots` | list every recorded boot |
| `journalctl _COMM=sshd` | filter by field match (`_COMM` = trusted field, command name) |
| `journalctl _COMM=sshd + _COMM=autossh` | OR two field matches with `+` |
| `journalctl --field=_COMM` | list every value a field can take |
| `journalctl -o verbose` | show every field of an entry (including trusted ones) |
| `journalctl -o json` / `-o json -f` | JSON output, can be combined with follow |
| `journalctl -o short-monotonic` | show the timestamp as time elapsed since boot |
| `journalctl --header` | show the journal file's own header info (format, size limits...) |

## Notes

| Concept | Note |
|---|---|
| `syslog` vs `rsyslog` | `syslog` is a protocol/API name (RFC 5424, the `syslog()` function), not a concrete program; `rsyslog` is Ubuntu's default concrete implementation (`r` = "rocket-fast", emphasizing performance) |
| journald ↔ rsyslog relationship | one-way copy (`ForwardToSyslog=yes`), two independent systems — one doesn't inherit the other's guarantees (see the 3.4 crash test) |
| `Storage=auto` (Ubuntu default) | writes to `/var/log/journal` if it exists (behaves like persistent), doesn't create the directory itself |
| `SyncIntervalSec` | the one durability setting; `Seal` is for tamper detection, `Compress` is for disk savings — neither affects durability |
| Rotation/retention (`SystemMaxUse`, `MaxRetentionSec`...) | not covered in this part, to be handled separately |

**Skipped:** disk quota was already covered in Part 4; rotation/retention deliberately left out (see above).

[↑ Go back to TOC](#table-of-contents)

---

# Part 4 — Users & groups

Machine: Ubuntu 24.04 (AWS). Main account `ubuntu`, test account `deneme`, service account `myapp`.

## Cheat sheet

| Command | What it does |
|---|---|
| `id [user]` | UID, primary GID, supplementary groups |
| `id -gn user` | primary group name only |
| `sudo adduser <user>` | user + group + home + skel copy |
| `sudo adduser --system --group --no-create-home <user>` | service account: UID<1000, nologin, no home |
| `sudo deluser <user>` / `--remove-home` | delete user / also delete home |
| `sudo deluser --system <user>` | delete service account (refuses without the flag) |
| `sudo groupadd <group>` / `sudo groupdel <group>` | create / delete group |
| `sudo usermod -aG <group> <user>` | **append** to supplementary group (without `-a` the list is replaced) |
| `sudo gpasswd -a <user> <group>` / `-d <user> <group>` | add to / remove from supplementary group |
| `sudo usermod -g <group> <user>` | change primary group (also moves files in home) |
| `sudo usermod -s <shell> <user>` | change login shell (`/usr/sbin/nologin` = disable) |
| `sudo usermod -e YYYY-MM-DD <user>` / `-e ''` | expire account on date / clear |
| `sudo passwd -l <user>` / `-u <user>` | lock / unlock password |
| `sudo chage -l <user>` / `-M 90 <user>` | list aging info / password lifetime 90 days |
| `newgrp <group>` | activate group without re-login (opens inner shell) |
| `su - <user>` / `su - <user> -c 'cmd'` | become `<user>` / run one command as `<user>` (`<user>`'s password) |
| `sudo -u <user> cmd` | run one command as `<user>` (your password, skips shell) |
| `sudo visudo -f /etc/sudoers.d/<user>` | write a sudoers rule (syntax-checked) |
| `sudo find / -uid <uid>` / `-gid <gid> 2>/dev/null` | find orphaned files |
| `sudo chgrp <group> file` | change a file's group |
| `cut -d: -f1 /etc/passwd` | all usernames |

## 4.1 — Identity

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)
```

| Field | Meaning |
|---|---|
| `uid` | Linux knows you by this number, not by name. 0 root, 1–999 system, 1000+ humans |
| `gid` | primary group. Files you create belong to it. Ubuntu gives every user a single-member group with their name |
| `groups` | supplementary groups. Most privileges come from here |

| Group | Grants |
|---|---|
| `sudo` | you can become root (the `%sudo` line in `/etc/sudoers`) |
| `adm` | you can read `/var/log`, nothing else |
| `lxd` | LXD container management (not Docker; Canonical's tool). Effectively root |
| `cdrom`, `dip` | legacy hardware groups, meaningless |

User and group work together: if you are the owner, your user decides; otherwise your group membership does. Admin habit is to manage via groups ("5 people should read logs" = add 5 users to `adm`).

## 4.2 — The four files

The whole user/group system is 4 text files. No database, no daemon. `adduser`, `usermod`, `gpasswd` are programs that edit them.

| File | Holds | Readable by |
|---|---|---|
| `/etc/passwd` | user identity: name, UID, primary GID, home, shell | everyone |
| `/etc/shadow` | user password hash + aging rules | root, `shadow` group |
| `/etc/group` | group identity: name, GID, **supplementary** members | everyone |
| `/etc/gshadow` | group password (practically always empty) | root |

Identity is public because `ls -l` must map UID→name. The hash is private because it can be cracked offline.

**`/etc/passwd`**

```
deneme : x : 1001 : 1001 : Jack Brown,31,n/a,n/a,n/a : /home/deneme : /bin/bash
 name    pw   UID   GID            GECOS                  home         shell
```

`x` = password lives in shadow. Shell `nologin` = login disabled but account exists.

**`/etc/shadow`**

```
deneme : $y$j9T$POUR... : 20716 : 0 : 99999 : 7 : : :
 name        hash         last   min   max  warn inactive expire
```

| Value | Meaning |
|---|---|
| `$y$` | yescrypt hash. The password itself is stored nowhere |
| leading `!` or `*` | locked. `ubuntu:!:...` → no password, logs in with SSH key |
| `20716` | days since 1970 |
| `99999` | never |

**`/etc/group`**

```
adm : x : 4 : syslog,ubuntu,deneme
name pw  GID   supplementary members
```

Primary members don't appear here; they're in the GID field of `passwd`. `deneme:x:1001:` empty = nobody is a supplementary member, but `deneme` is in it as primary.

## 4.3 — Creating a user

```
$ sudo adduser deneme
info: Adding new group `deneme' (1001) ...
info: Adding new user `deneme' (1001) with group `deneme (1001)' ...
info: Creating home directory `/home/deneme' ...
info: Copying files from `/etc/skel' ...
New password:
```

| Step | Where |
|---|---|
| user line | `/etc/passwd` |
| hash | `/etc/shadow` |
| single-member group | `/etc/group` |
| home | `/home/deneme`, `drwxr-x---` (others can't enter) |
| template | `/etc/skel` → copied to home, ownership to the user |

`/etc/skel` = skeleton: `.bashrc`, `.profile`, `.bash_logout`. Anything placed here goes into every future user.

## 4.4 — `ls -l` format

```
-rw-r-----  1  syslog  adm  313512  Sep 20 22:00  /var/log/syslog
   perms   links owner  group  size      date          name
```

Permissions are 3 blocks: **owner / group / others** (`u`/`g`/`o`). First character is the type: `-` file, `d` directory, `l` symlink.

| Block | For `syslog` | Who |
|---|---|---|
| `rw-` | read, write | owner `syslog` (rsyslog writes as this identity) |
| `r--` | read | group `adm` |
| `---` | nothing | others |

`syslog` is a service account: `syslog:x:102:102::/nonexistent:/usr/sbin/nologin`. Writer and readers are separated.

## 4.5 — Privilege through groups

`deneme` is not in `adm` → others → `---`:

```
$ su - deneme -c 'head -3 /var/log/syslog'
cat: /var/log/syslog: Permission denied
```

Don't touch the file; add the user to the group:

```
$ sudo usermod -aG adm deneme
$ su - deneme -c 'head -3 /var/log/syslog'
2026-09-20T18:21:48.060807+00:00 ip-172-31-78-215 kernel: Linux version 6.17.0-1017-aws ...
```

| Task | `usermod` | `gpasswd` |
|---|---|---|
| add | `sudo usermod -aG adm deneme` | `sudo gpasswd -a deneme adm` |
| remove | `sudo usermod -G users,gizli deneme` (rewrite the whole list) | `sudo gpasswd -d deneme adm` |
| order | flag → group → user | flag → user → group |
| danger | forget `-a` and the list is replaced | none |

⚠️ Group changes don't affect open sessions; re-login or `newgrp` (4.9).

**`su` vs `sudo`**

| | `sudo` | `su` |
|---|---|---|
| does | run one command as someone else | open a shell as someone else |
| password | yours | the target's |
| permission | sudoers | knowing the target's password |
| restrictable | yes, per command | no |
| target is `nologin` | works (skips the shell) | fails |

`su - <user>`: `-` = login shell, rebuild <user>'s environment from scratch. Don't use it without the dash. `-c 'cmd'` = no shell, run the command and exit.

## 4.6 — Who may run a program

**Way 1: program runs with user privileges → file group + `chmod 750`**

```
$ sudo groupadd gizli
$ printf '#!/bin/bash\necho "gizli program calisti, ben: $(whoami)"\n' | sudo tee /usr/local/bin/gizli-program
$ sudo chmod 750 /usr/local/bin/gizli-program
$ sudo chown root:gizli /usr/local/bin/gizli-program
$ ls -l /usr/local/bin/gizli-program
-rwxr-x--- 1 root gizli 57 /usr/local/bin/gizli-program
```

| Who | Result | Why |
|---|---|---|
| `ubuntu` | denied | not in `gizli` |
| `sudo` | `ben: root` | permissions don't apply to root |
| `deneme` | denied | not in `gizli` |
| `deneme` (after `usermod -aG gizli`) | `ben: deneme` | joined the group |

Same mechanism as the log file, `x` instead of `r`.

**Way 2: program needs root → per-command rule in sudoers**

```
deneme@lev-k:~$ sudo cat /etc/shadow
deneme is not in the sudoers file.
```

```
$ sudo visudo -f /etc/sudoers.d/deneme
```
```
deneme  ALL=(root)  /usr/bin/cat /etc/shadow
```

Read it as a sentence: **WHO, WHERE = (AS WHOM) WHAT**. `ALL` = any host; the format requires it.

```
deneme@lev-k:~$ sudo cat /etc/shadow
root:*:20614:0:99999:7:::                    # works
deneme@lev-k:~$ sudo cat /etc/passwd
Sorry, user deneme is not allowed to execute '/usr/bin/cat /etc/passwd' as root on lev-k.
```

Same `cat`, different argument, denied. sudoers matches the command **with its arguments**.

| sudoers | Note |
|---|---|
| `/etc/sudoers` | main file, last line `@includedir /etc/sudoers.d` |
| `/etc/sudoers.d/*` | extras, appended to the same list. Just for tidiness |
| `visudo` | locks, syntax-checks, refuses to save a broken file. Broken sudoers = `sudo` locks up |
| `%sudo ALL=(ALL:ALL) ALL` | `%` = group. Ubuntu's `sudo` group gets its power here |
| `ubuntu ALL=(ALL) NOPASSWD:ALL` | written by cloud-init, `/etc/sudoers.d/90-cloud-init-users`. Why no password prompt |
| reader | the `sudo` command itself, every run. No daemon |

## 4.7 — Deleting groups and orphaned GIDs

```
$ sudo gpasswd -d deneme adm
Removing user deneme from group adm
$ sudo groupdel gizli                          # supplementary members drop automatically; refuses if it's someone's primary
$ ls -l /usr/local/bin/gizli-program
-rwxr-x--- 1 root 1002 57 /usr/local/bin/gizli-program
```

Group gone, file still has GID 1002, `ls` can't map it to a name. Danger: a new group that gets 1002 inherits the file.

```
$ sudo find / -gid 1002 2>/dev/null
/usr/local/bin/gizli-program
$ sudo chgrp root /usr/local/bin/gizli-program
```

`2>/dev/null` = swallow stderr (`find` chases its own tail in `/proc`, noise). Right order: **`find` first, then `groupdel`/`deluser`**.

## 4.8 — Primary group

One job: the group of files you create. Supplementary = "where can I reach", primary = "what I create belongs to whom".

```
$ sudo usermod -g gizli deneme                 # lowercase -g = primary
$ grep '^deneme:' /etc/passwd
deneme:x:1001:1002:...                         # GID 1001 → 1002
```

⚠️ `usermod -g` also moves files **in home** owned by the old primary group to the new one. Doesn't touch anything outside home. Revert: `sudo usermod -g deneme deneme`.

Umask note: if primary group name = username, umask is `002` (`-rw-rw-r--`), otherwise `022` (`-rw-r--r--`). Part 9.

## 4.9 — `newgrp`: group without re-login

The shell copies the group list **at login**. `usermod` changes the file, not the open shell:

```
deneme@lev-k:~$ cat /etc/group | grep gizli
gizli:x:1002:deneme                            # member in the file
deneme@lev-k:~$ id
uid=1001(deneme) gid=1001(deneme) groups=1001(deneme),100(users)   # not in the shell
deneme@lev-k:~$ gizli-program
-bash: /usr/local/bin/gizli-program: Permission denied
```

```
deneme@lev-k:~$ newgrp gizli
deneme@lev-k:~$ gizli-program
gizli program calisti, ben: deneme
deneme@lev-k:~$ id
uid=1001(deneme) gid=1002(gizli) groups=1002(gizli),100(users),1001(deneme)
```

| `newgrp <group>` | |
|---|---|
| does | opens an inner shell (`$SHLVL` 1→2), adds `<group>` **and makes it primary** |
| requires | membership in `/etc/group`; otherwise asks for a group password (none) → denied |
| permanent | no, `exit` returns to the old shell |
| `sg <group> -c 'cmd'` | one command without opening a shell |

```
deneme@lev-k:~$ touch test1.txt               # inner shell → group gizli
deneme@lev-k:~$ exit
deneme@lev-k:~$ touch test2.txt               # outer shell → group deneme
-rw-rw-r-- 1 deneme gizli  0 test1.txt
-rw-rw-r-- 1 deneme deneme 0 test2.txt
```

Process tree:
```
su (root) → -bash (deneme, login) → newgrp → bash (deneme, gizli active)
```

## 4.10 — Restricting users

Cut access without deleting. Each one changes a field in `passwd`/`shadow`.

| Command | Blocks | Still open | Scenario |
|---|---|---|---|
| `passwd -l <user>` | password login (`!` before hash) | SSH key, cron, running processes | leave, temporary suspension |
| `usermod -s /usr/sbin/nologin <user>` | every interactive login | cron, running processes | permanent shutdown, service accounts |
| `usermod -e 2026-12-31 <user>` | everything after the date | everything until then | intern, temporary access |
| `chage -M 90 <user>` | forces password change after 90 days | everything | password policy |

| Revert | |
|---|---|
| `passwd -u <user>` | unlock, old password works |
| `usermod -s /bin/bash <user>` | restore shell |
| `usermod -e '' <user>` | clear expiry |
| `chage -l <user>` | show all aging info, human-readable |

Error messages differ and tell you where it stopped:

```
$ su - deneme                                  # after passwd -l
su: Authentication failure                     # password stage
$ su - deneme                                  # after nologin
This account is currently not available.       # shell stage
$ su - deneme                                  # after usermod -e 2026-01-01
Your account has expired; please contact your system administrator.
```

The shell field can be any program (kiosk menu, `git-shell`, `rrsync`):

```
$ printf '#!/bin/bash\necho "Buraya giris yok canim :)"\n' | sudo tee /usr/local/bin/giris-yok
$ sudo chmod 755 /usr/local/bin/giris-yok
$ sudo usermod -s /usr/local/bin/giris-yok deneme
$ su - deneme
Password:
Buraya giris yok canim :)
```

## 4.11 — Service accounts

Don't run applications as root; if hacked, the attacker is root. Give the app its own user that can't log in and has no home. `syslog`, `sshd`, `www-data` are like this. Kubernetes `runAsUser` is the same idea.

```
$ sudo adduser --system --group --no-create-home myapp
$ grep -E '^(syslog|myapp):' /etc/passwd
syslog:x:102:102::/nonexistent:/usr/sbin/nologin
myapp:x:111:113::/nonexistent:/usr/sbin/nologin
```

| Flag | Does |
|---|---|
| `--system` | UID 100–999, shell `nologin`, no password or GECOS prompts |
| `--group` | same-named group, make it primary (otherwise `nogroup`) |
| `--no-create-home` | no `/home/myapp`; working dir becomes `/var/lib/myapp` |
| `--uid 5000` | overrides the range; breaks the "below 1000 = service" convention, avoid unless needed |

The UID range is convention, not a kernel limit (OpenShift uses 1000000000+). Using it: no login, no password, `su` fails → `sudo -u myapp cmd` or systemd `User=myapp`.

```
$ sudo -u myapp whoami
myapp
$ sudo -u myapp gizli-program
sudo: unable to execute /usr/local/bin/gizli-program: Permission denied    # zero privilege, correct start
```

### Case study: the `myapp` service

Goal: a script writes the date to `/var/lib/myapp/log.txt` every second, systemd runs it as `myapp`, everyone can read, nobody can write, only `myapp` can execute the script.

| Step | Command |
|---|---|
| account | `sudo adduser --system --group --no-create-home myapp` |
| directory | `sudo mkdir /var/lib/myapp && sudo chown myapp:myapp /var/lib/myapp && sudo chmod 755 /var/lib/myapp` |
| script | `sudo nano /usr/local/bin/myapp-logger` |
| ownership | `sudo chown myapp:myapp /usr/local/bin/myapp-logger && sudo chmod 700 /usr/local/bin/myapp-logger` |
| manual test | `sudo -u myapp timeout 25 /usr/local/bin/myapp-logger` |
| unit | `sudo nano /etc/systemd/system/myapp.service` |
| start | `sudo systemctl daemon-reload && sudo systemctl start myapp` |
| check | `systemctl status myapp`, `ps -o user,pid,cmd -C myapp-logger` |
| watch | `tail -f /var/lib/myapp/log.txt` |

Script:
```bash
#!/bin/bash
while true; do
  date >> /var/lib/myapp/log.txt
  sleep 1
done
```

Unit:
```ini
[Unit]
Description=myapp logger

[Service]
User=myapp
ExecStart=/usr/local/bin/myapp-logger
Restart=always

[Install]
WantedBy=multi-user.target
```

| systemd | Meaning |
|---|---|
| `User=myapp` | the service runs as this identity |
| `Restart=always` | restart on crash; `systemctl stop` doesn't count |
| `[Install] WantedBy=` | when to start at boot, used by `enable` |
| `daemon-reload` | re-read unit files; doesn't touch running services. After every unit edit |
| `start` / `stop` / `restart` | start now / stop / stop+start |
| `enable` / `enable --now` | start at boot / + start now too |

Process:

1. `chmod 700` + `chown myapp` → `ubuntu` can't run it: `Permission denied`. Root can; permissions don't apply to root.
2. Manual test: `timeout 25` kills the infinite loop after 25 s, the terminal doesn't hang.
3. `deneme` reads, can't write:
   ```
   deneme@lev-k:~$ tail -f /var/lib/myapp/log.txt
   Mon Sep 21 12:43:47 UTC 2026
   deneme@lev-k:~$ echo hack >> /var/lib/myapp/log.txt
   -bash: /var/lib/myapp/log.txt: Permission denied
   ```
4. The service runs as `myapp`:
   ```
   $ ps -o user,pid,cmd -C myapp-logger
   USER         PID CMD
   myapp       3395 /bin/bash /usr/local/bin/myapp-logger
   ```
5. `systemctl stop` without `sudo` → polkit asks for `ubuntu`'s password, there is none, denied. With `sudo` it consults sudoers and passes.

Cleanup: `stop` → `rm unit` → `daemon-reload` → `deluser --system myapp` → `rm -rf /var/lib/myapp` → `rm script`.

## 4.12 — Deleting a user

| Command | Home | Other files |
|---|---|---|
| `sudo deluser <user>` | kept | untouched |
| `sudo deluser --remove-home <user>` | deleted | untouched |

Files elsewhere are orphaned by UID; a new user that gets the same UID inherits them.

```
$ sudo -u deneme touch /tmp/deneme-dosyasi
$ sudo deluser --remove-home deneme
userdel: user deneme is currently used by process 3793
```

A user with a running process can't be deleted. `ps -o user,pid,cmd -p 3793` → `deneme -bash`. Interactive bash ignores SIGTERM (so you don't lose work by accident); `kill -9` is needed:

```
$ sudo kill -9 3793
$ sudo deluser --remove-home deneme
$ ls -l /tmp/deneme-dosyasi
-rw-rw-r-- 1 1001 1001 0 /tmp/deneme-dosyasi       # no name, just the UID
$ sudo find / -uid 1001 2>/dev/null
/tmp/deneme-dosyasi
```

Right order: **`find -uid` → delete/`chown` → `deluser`**.

## Notes

**`usermod` flags** (user modify, from the `passwd` package on Ubuntu):

| Flag | Word | Field |
|---|---|---|
| `-s` | shell | passwd 7 |
| `-g` | group | passwd 4 (primary) |
| `-aG` | append groups | `/etc/group` |
| `-e` | expire | shadow 8 |
| `-d` | directory | passwd 6 |
| `-l` | login | passwd 1 (rename) |
| `-L` / `-U` | lock / unlock | = `passwd -l/-u` |

**`sudo` and `>`:** `sudo echo x > /root/f` fails; the shell performs `>` with *your* privileges, `sudo` only applies to `echo`. `tee` is a program, so `sudo` runs it as root: `echo x | sudo tee /root/f`. Same reason `sudo rm /home/deneme/test*.txt` fails (your shell expands `*` and can't enter the directory): `sudo sh -c 'rm /home/deneme/test*.txt'`.

**`/run/sudo/ts/UID`:** `sudo`'s 15-minute "don't ask again" memory. `sudo -k` resets it.

**Skipped:** disk quota (off on Ubuntu; for multi-user shared systems).

[↑ Go back to TOC](#table-of-contents)

---

# Part 5 — Process management

> 🔜 Placeholder — ps, top/htop, signals (SIGTERM vs SIGKILL), nice/renice.

[↑ Go back to TOC](#table-of-contents)

---

# Part 6 — Package management (apt)

> 🔜 Placeholder — how apt works: repositories, sources lists, GPG keys, installing binaries, updates.

[↑ Go back to TOC](#table-of-contents)

---

# Part 7 — SSH & sshd

> 🔜 Placeholder — sshd setup, key management, sshd_config, hardening.

[↑ Go back to TOC](#table-of-contents)

---

# Part 8 — Networking basics

> 🔜 Placeholder — ip, ss, ping, DNS, firewall (ufw → nftables), network namespaces.

[↑ Go back to TOC](#table-of-contents)

---

# Part 9 — File permissions & ownership

> 🔜 Placeholder — chmod, chown, umask, setuid/setgid, ACL.

[↑ Go back to TOC](#table-of-contents)

---

# Part 10 — Disk & filesystem

> 🔜 Placeholder — mount, fstab, lsblk, df/du, intro to LVM.

[↑ Go back to TOC](#table-of-contents)

---

# Part 11 — Cron & timers

> 🔜 Placeholder — cron vs systemd timers.

[↑ Go back to TOC](#table-of-contents)

---

# Part 12 — Memory & I/O

> 🔜 Placeholder — page cache, dirty pages, background writeback, `fsync`/`fdatasync`, `/proc/sys/vm/*`. Split out of Process management (Part 5) into its own section.

[↑ Go back to TOC](#table-of-contents)
