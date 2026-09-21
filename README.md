<div align="right">

**English** | [Türkçe](README.tr.md)

</div>

# Linux Sysadmin Lab

A hands-on Linux system administration lab: systemd, users & groups, apt, logging, processes, SSH and networking — learning how a Linux system actually works by poking at it, not by reading about it.

All experiments run on a real **Ubuntu** machine, and the outputs shown are real, not idealized.

The longer-term goal behind this lab is virtualization and Kubernetes. Almost every chapter here is a building block for that world: systemd runs the kubelet, users and permissions become security contexts, veth pairs and bridges are how pods talk to each other. This lab is the Linux underneath Kubernetes.

## Table of contents

- [Part 0 — Everything is a file](#part-0--everything-is-a-file)
- [Part 1 — systemd & systemctl](#part-1--systemd--systemctl)
- [Part 2 — Logging & journalctl](#part-2--logging--journalctl)
- [Part 3 — Users & groups](#part-3--users--groups)
- [Part 4 — Package management (apt)](#part-4--package-management-apt)
- [Part 5 — Process management](#part-5--process-management)
- [Part 6 — SSH & sshd](#part-6--ssh--sshd)
- [Part 7 — Networking basics](#part-7--networking-basics)
- [Part 8 — File permissions & ownership](#part-8--file-permissions--ownership)
- [Part 9 — Disk & filesystem](#part-9--disk--filesystem)
- [Part 10 — Cron & timers](#part-10--cron--timers)

## Curriculum

| # | Part | Question it answers | Status |
|---|------|--------------------|--------|
| 0 | [Everything is a file](#part-0--everything-is-a-file) | What is a file, a file descriptor, a socket — and why is *everything* one? | ⏳ |
| 1 | [systemd & systemctl](#part-1--systemd--systemctl) | How does systemd control every program on the machine? | 🔜 |
| 2 | [Logging & journalctl](#part-2--logging--journalctl) | Where do logs live, and how do I interrogate the journal? | 🔜 |
| 3 | [Users & groups](#part-3--users--groups) | How do I create, restrict and destroy users — and what is a group really? | ✅ |
| 4 | [Package management (apt)](#part-4--package-management-apt) | What actually happens on `apt install` — repos, GPG keys, binaries? | 🔜 |
| 5 | [Process management](#part-5--process-management) | What is a process, a signal — and what really separates SIGTERM from SIGKILL? | 🔜 |
| 6 | [SSH & sshd](#part-6--ssh--sshd) | How do I set up and harden sshd, and manage keys properly? | 🔜 |
| 7 | [Networking basics](#part-7--networking-basics) | How does the machine talk — interfaces, DNS, firewall, namespaces? | 🔜 |
| 8 | [File permissions & ownership](#part-8--file-permissions--ownership) | Who may touch what — chmod, umask, setuid, ACL? | 🔜 |
| 9 | [Disk & filesystem](#part-9--disk--filesystem) | How do disks become directories — mount, fstab, LVM? | 🔜 |
| 10 | [Cron & timers](#part-10--cron--timers) | How do I run things on a schedule — and cron vs systemd timers? | 🔜 |

---

# Part 0 — Everything is a file

> ⏳ Placeholder — files, inodes, file descriptors, special files (device, pipe, socket), `lsof`, `/proc/PID/fd`.

[↑ Go back to TOC](#table-of-contents)

---

# Part 1 — systemd & systemctl

> 🔜 Placeholder — how systemd controls programs: units, targets, service lifecycle, writing a unit file.

[↑ Go back to TOC](#table-of-contents)

---

# Part 2 — Logging & journalctl

> 🔜 Placeholder — the journal, filtering by unit/time/priority, rsyslog, log rotation.

[↑ Go back to TOC](#table-of-contents)

---

# Part 3 — Users & groups

Machine: Ubuntu 24.04 (AWS), user `ubuntu`. Test user: `deneme`.

## 3.1 — Who am I?

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)
```

- Linux knows you by **UID**, not by name. 0 = root, 1–999 system accounts, 1000+ humans.
- `gid` = primary group. Ubuntu creates a group with your name for every user.
- `groups` = supplementary groups. Most privileges come from here:
  - `sudo` → you can become root
  - `adm` → you can read `/var/log`
  - `lxd` → you can manage LXD containers (not Docker; Canonical's tool; effectively root)

## 3.2 — Creating a user

```
$ sudo adduser deneme
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `deneme' (1001) ...
info: Adding new user `deneme' (1001) with group `deneme (1001)' ...
info: Creating home directory `/home/deneme' ...
info: Copying files from `/etc/skel' ...
New password:
```

`adduser` really does 5 things:

| What | Where |
|---|---|
| user line | `/etc/passwd` |
| password hash | `/etc/shadow` |
| same-named group | `/etc/group` |
| home directory | `/home/deneme` (`drwxr-x---`, others can't enter) |
| template files | `/etc/skel` → copied to home |

**`/etc/skel`** = skeleton. Contains `.bashrc`, `.profile`, `.bash_logout`. Anything you put here gets copied into every user created from now on.

### `/etc/passwd` — the user list

A user *is* a line in this file. No folder, one file.

```
$ grep -E '^(ubuntu|deneme):' /etc/passwd
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
deneme:x:1001:1001:Jack Brown,31,n/a,n/a,n/a:/home/deneme:/bin/bash
```

```
deneme : x : 1001 : 1001 : Jack Brown,... : /home/deneme : /bin/bash
 name   pw   UID   GID      GECOS            home          shell
```

`x` = "password isn't here, look in shadow". Set the shell to `/usr/sbin/nologin` and the user can't log in but still exists (service accounts work this way).

### `/etc/shadow` — passwords

```
$ sudo grep -E '^(ubuntu|deneme):' /etc/shadow
ubuntu:!:20716:0:99999:7:::
deneme:$y$j9T$POUR...:20716:0:99999:7:::
```

- `$y$` = yescrypt hash. The password itself is stored nowhere.
- `!` = password **locked**. `ubuntu` has no password; it logs in with an SSH key.
- Remaining fields: last change (days since epoch), min days, max days, warning.

Why `sudo` was needed:

```
$ ls -l /etc/passwd /etc/shadow
-rw-r--r-- 1 root root   1918 /etc/passwd
-rw-r----- 1 root shadow  999 /etc/shadow
```

`passwd` is world-readable (`ls` reads it to map UID → name). `shadow` is readable only by root and the `shadow` group.

## 3.3 — Granting privilege through groups

Can `deneme` read logs?

```
$ su - deneme -c 'head -3 /var/log/syslog'
cat: /var/log/syslog: Permission denied

$ ls -l /var/log/syslog
-rw-r----- 1 syslog adm 313137 /var/log/syslog
```

`ls -l` format:

```
-rw-r-----  1  syslog  adm  313137  Sep 20 22:00  /var/log/syslog
   perms   links owner  group  size      date          name
```

Permissions are 3 blocks: **owner / group / others** (`u`/`g`/`o`).
- `rw-` owner (`syslog`): read, write
- `r--` group (`adm`): read
- `---` others: nothing

`syslog` is a service account (`syslog:x:102:102::/nonexistent:/usr/sbin/nologin`). rsyslog runs as this identity and writes the log. Read access goes to the `adm` group. Writer and readers are separated.

`deneme` is not the owner, not in `adm` → others → `---`. Fix: add to the group.

```
$ sudo usermod -aG adm deneme
$ su - deneme -c 'head -3 /var/log/syslog'
2026-09-20T18:21:48.060807+00:00 ip-172-31-78-215 kernel: Linux version 6.17.0-1017-aws ...
```

We never touched the file. We only changed the user's group membership.

> ⚠️ `usermod -G` on its own **replaces** the supplementary group list. Always use `-aG` (append).

> ⚠️ Group changes don't affect open sessions; log out and back in.

**`su` notes:**
- `su - deneme` → become `deneme`. `-` = login shell: rebuild their environment from scratch (home, `$PATH`, `.profile`). Don't use it without the dash.
- `-c 'cmd'` → don't open a shell, run the command and exit.
- `su` asks for the **target** user's password; `sudo` asks for **yours** and consults sudoers.

## 3.4 — Who may run a program?

### Way 1: The program runs with user privileges → group + `chmod 750`

```
$ sudo groupadd gizli
$ printf '#!/bin/bash\necho "gizli program calisti, ben: $(whoami)"\n' | sudo tee /usr/local/bin/gizli-program
$ sudo chmod 750 /usr/local/bin/gizli-program
$ sudo chown root:gizli /usr/local/bin/gizli-program
$ ls -l /usr/local/bin/gizli-program
-rwxr-x--- 1 root gizli 57 /usr/local/bin/gizli-program
```

Test:

```
$ gizli-program
-bash: /usr/local/bin/gizli-program: Permission denied      # ubuntu, not in gizli

$ sudo gizli-program
gizli program calisti, ben: root                             # permissions don't apply to root

$ su - deneme -c 'gizli-program'
-bash: line 1: /usr/local/bin/gizli-program: Permission denied

$ sudo usermod -aG gizli deneme
$ su - deneme -c 'gizli-program'
gizli program calisti, ben: deneme
```

Same mechanism as the log file, `x` instead of `r`.

### Way 2: The program needs root → per-command permission in sudoers

`deneme` can't become root:

```
deneme@lev-k:~$ sudo cat /etc/shadow
deneme is not in the sudoers file.
```

Let them run only this one command as root. Use `visudo` (it syntax-checks and refuses to save a broken file; otherwise `sudo` locks up entirely):

```
$ sudo visudo -f /etc/sudoers.d/deneme
```

One line inside:

```
deneme  ALL=(root)  /usr/bin/cat /etc/shadow
```

Read it as a sentence: **WHO, WHERE = (AS WHOM) WHAT**. `ALL` = any host; the format requires it.

Test:

```
deneme@lev-k:~$ sudo cat /etc/shadow
root:*:20614:0:99999:7:::
...                                                          # works

deneme@lev-k:~$ sudo cat /etc/passwd
Sorry, user deneme is not allowed to execute '/usr/bin/cat /etc/passwd' as root on lev-k.
```

Same `cat`, different argument, denied. sudoers matches the command **together with its arguments**.

**How sudoers works:**
- One source: `/etc/sudoers`. Its last line is `@includedir /etc/sudoers.d` → files in that folder are appended to the same list. The folder is just for tidiness.
- The reader is the `sudo` command itself, from scratch on every run. No service, no daemon.
- `%sudo ALL=(ALL:ALL) ALL` → `%` = group. Ubuntu's `sudo` group gets its power from this line.
- On this machine `ubuntu`'s power actually comes from a file written by cloud-init:

```
$ sudo cat /etc/sudoers.d/90-cloud-init-users
ubuntu ALL=(ALL) NOPASSWD:ALL
```

`NOPASSWD` → no password prompt. That's why `sudo` never asked for one.

### `sudo` vs `su`

| | `sudo` | `su` |
|---|---|---|
| What it does | Run one command as someone else | Open a shell as someone else |
| Whose password | Yours | The target's |
| Where permission comes from | sudoers | Knowing the target's password |
| Restrictable? | Yes, per command | No |

## Skipped

- **Disk quota** — off by default on Ubuntu; meant for multi-user shared systems. Revisit when needed.
- Why `tee` instead of `>` — `sudo echo > file` fails because the shell does the `>` with *your* privileges; `tee` is a program, so `sudo` runs it as root. To be covered separately.

[↑ Go back to TOC](#table-of-contents)

---

# Part 4 — Package management (apt)

> 🔜 Placeholder — how apt works: repositories, sources lists, GPG keys, installing binaries, updates.

[↑ Go back to TOC](#table-of-contents)

---

# Part 5 — Process management

> 🔜 Placeholder — ps, top/htop, signals (SIGTERM vs SIGKILL), nice/renice.

[↑ Go back to TOC](#table-of-contents)

---

# Part 6 — SSH & sshd

> 🔜 Placeholder — sshd setup, key management, sshd_config, hardening.

[↑ Go back to TOC](#table-of-contents)

---

# Part 7 — Networking basics

> 🔜 Placeholder — ip, ss, ping, DNS, firewall (ufw → nftables), network namespaces.

[↑ Go back to TOC](#table-of-contents)

---

# Part 8 — File permissions & ownership

> 🔜 Placeholder — chmod, chown, umask, setuid/setgid, ACL.

[↑ Go back to TOC](#table-of-contents)

---

# Part 9 — Disk & filesystem

> 🔜 Placeholder — mount, fstab, lsblk, df/du, intro to LVM.

[↑ Go back to TOC](#table-of-contents)

---

# Part 10 — Cron & timers

> 🔜 Placeholder — cron vs systemd timers.

[↑ Go back to TOC](#table-of-contents)
