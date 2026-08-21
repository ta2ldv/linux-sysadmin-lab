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
| 3 | [Users & groups](#part-3--users--groups) | How do I create, restrict and destroy users — and what is a group really? | 🔜 |
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

> 🔜 Placeholder — creating and deleting users, restricting them, group logic, `/etc/passwd`, `/etc/shadow`, sudo.

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
