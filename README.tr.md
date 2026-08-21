<div align="right">

[English](README.md) | **Türkçe**

</div>

# Linux Sysadmin Lab

Uygulamalı bir Linux system administration lab'ı: systemd, user & group, apt, logging, process'ler, SSH ve networking — Linux'un gerçekte nasıl çalıştığını okuyarak değil, kurcalayarak öğrenmek.

Tüm deneyler gerçek bir **Ubuntu** makinesinde koşuyor; gösterilen çıktılar gerçek, idealize edilmiş değil.

Bu lab'ın uzun vadeli hedefi virtualization ve Kubernetes. Buradaki hemen her bölüm o dünyanın yapı taşı: kubelet'i systemd çalıştırıyor, user ve permission'lar security context'e dönüşüyor, pod'lar birbiriyle veth pair ve bridge üzerinden konuşuyor. Bu lab, Kubernetes'in altındaki Linux.

## İçindekiler

- [Bölüm 0 — Everything is a file](#bölüm-0--everything-is-a-file)
- [Bölüm 1 — systemd & systemctl](#bölüm-1--systemd--systemctl)
- [Bölüm 2 — Logging & journalctl](#bölüm-2--logging--journalctl)
- [Bölüm 3 — Users & groups](#bölüm-3--users--groups)
- [Bölüm 4 — Package management (apt)](#bölüm-4--package-management-apt)
- [Bölüm 5 — Process management](#bölüm-5--process-management)
- [Bölüm 6 — SSH & sshd](#bölüm-6--ssh--sshd)
- [Bölüm 7 — Networking temelleri](#bölüm-7--networking-temelleri)
- [Bölüm 8 — File permissions & ownership](#bölüm-8--file-permissions--ownership)
- [Bölüm 9 — Disk & filesystem](#bölüm-9--disk--filesystem)
- [Bölüm 10 — Cron & timers](#bölüm-10--cron--timers)

## Müfredat

| # | Bölüm | Cevapladığı soru | Durum |
|---|-------|------------------|-------|
| 0 | [Everything is a file](#bölüm-0--everything-is-a-file) | File nedir, file descriptor nedir, socket nedir — ve neden *her şey* bir file? | ⏳ |
| 1 | [systemd & systemctl](#bölüm-1--systemd--systemctl) | systemd makinedeki her programı nasıl kontrol ediyor? | 🔜 |
| 2 | [Logging & journalctl](#bölüm-2--logging--journalctl) | Log'lar nerede yaşıyor, journal nasıl sorgulanır? | 🔜 |
| 3 | [Users & groups](#bölüm-3--users--groups) | User nasıl yaratılır, kısıtlanır, yok edilir — group gerçekte nedir? | 🔜 |
| 4 | [Package management (apt)](#bölüm-4--package-management-apt) | `apt install` deyince gerçekte ne oluyor — repo'lar, GPG key'ler, binary'ler? | 🔜 |
| 5 | [Process management](#bölüm-5--process-management) | Process nedir, signal nedir — SIGTERM ile SIGKILL'i gerçekte ne ayırır? | 🔜 |
| 6 | [SSH & sshd](#bölüm-6--ssh--sshd) | sshd nasıl kurulur ve hardening yapılır, key'ler nasıl yönetilir? | 🔜 |
| 7 | [Networking temelleri](#bölüm-7--networking-temelleri) | Makine nasıl konuşuyor — interface'ler, DNS, firewall, namespace'ler? | 🔜 |
| 8 | [File permissions & ownership](#bölüm-8--file-permissions--ownership) | Kim neye dokunabilir — chmod, umask, setuid, ACL? | 🔜 |
| 9 | [Disk & filesystem](#bölüm-9--disk--filesystem) | Disk nasıl directory'ye dönüşüyor — mount, fstab, LVM? | 🔜 |
| 10 | [Cron & timers](#bölüm-10--cron--timers) | Bir işi zamanlayarak nasıl koştururum — cron mu systemd timer mı? | 🔜 |

---

# Bölüm 0 — Everything is a file

> ⏳ Placeholder — file, inode, file descriptor, special file'lar (device, pipe, socket), `lsof`, `/proc/PID/fd`.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 1 — systemd & systemctl

> 🔜 Placeholder — systemd programları nasıl kontrol eder: unit'ler, target'lar, service lifecycle, unit file yazmak.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 2 — Logging & journalctl

> 🔜 Placeholder — journal, unit/zaman/priority ile filtreleme, rsyslog, log rotation.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 3 — Users & groups

> 🔜 Placeholder — user yaratma ve silme, kısıtlama, group mantığı, `/etc/passwd`, `/etc/shadow`, sudo.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 4 — Package management (apt)

> 🔜 Placeholder — apt nasıl çalışır: repository'ler, sources list, GPG key'ler, binary kurulumu, update.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 5 — Process management

> 🔜 Placeholder — ps, top/htop, signal'lar (SIGTERM vs SIGKILL), nice/renice.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 6 — SSH & sshd

> 🔜 Placeholder — sshd kurulumu, key management, sshd_config, hardening.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 7 — Networking temelleri

> 🔜 Placeholder — ip, ss, ping, DNS, firewall (ufw → nftables), network namespace'ler.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 8 — File permissions & ownership

> 🔜 Placeholder — chmod, chown, umask, setuid/setgid, ACL.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 9 — Disk & filesystem

> 🔜 Placeholder — mount, fstab, lsblk, df/du, LVM'e giriş.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 10 — Cron & timers

> 🔜 Placeholder — cron vs systemd timer.

[↑ İçindekilere dön](#i̇çindekiler)
