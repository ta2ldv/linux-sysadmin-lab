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
| 3 | [Users & groups](#bölüm-3--users--groups) | User nasıl yaratılır, kısıtlanır, yok edilir — group gerçekte nedir? | ✅ |
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

Makine: Ubuntu 24.04 (AWS), user `ubuntu`. Deneme user'ı: `deneme`.

## 3.1 — Ben kimim?

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)
```

- Linux seni isimle değil **UID** ile tanır. 0 = root, 1–999 sistem hesapları, 1000+ insanlar.
- `gid` = primary group. Ubuntu her user'a kendi adıyla bir group açar.
- `groups` = üye olduğun ek group'lar. Yetkiler çoğunlukla buradan gelir:
  - `sudo` → root olabilirsin
  - `adm` → `/var/log` okuyabilirsin
  - `lxd` → LXD container yönetebilirsin (Docker değil, Canonical'ın aracı; pratikte root'a eşdeğer)

## 3.2 — User yaratmak

```
$ sudo adduser deneme
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `deneme' (1001) ...
info: Adding new user `deneme' (1001) with group `deneme (1001)' ...
info: Creating home directory `/home/deneme' ...
info: Copying files from `/etc/skel' ...
New password:
```

`adduser` aslında 5 iş yapar:

| İş | Nereye |
|---|---|
| user satırı | `/etc/passwd` |
| şifre hash'i | `/etc/shadow` |
| aynı isimde group | `/etc/group` |
| home dizini | `/home/deneme` (`drwxr-x---`, others giremez) |
| şablon dosyalar | `/etc/skel` → home'a kopya |

**`/etc/skel`** = skeleton. İçinde `.bashrc`, `.profile`, `.bash_logout` var. Buraya koyduğun her şey bundan sonra yaratılan her user'ın home'una kopyalanır.

### `/etc/passwd` — user listesi

User demek bu dosyada bir satır demek. Folder yok, tek dosya.

```
$ grep -E '^(ubuntu|deneme):' /etc/passwd
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
deneme:x:1001:1001:Jack Brown,31,n/a,n/a,n/a:/home/deneme:/bin/bash
```

```
deneme : x : 1001 : 1001 : Jack Brown,... : /home/deneme : /bin/bash
 isim   şifre UID   GID      GECOS            home          shell
```

`x` = "şifre burada değil, shadow'a bak". Shell'i `/usr/sbin/nologin` yaparsan user login yapamaz ama var olmaya devam eder (servis hesapları böyle).

### `/etc/shadow` — şifreler

```
$ sudo grep -E '^(ubuntu|deneme):' /etc/shadow
ubuntu:!:20716:0:99999:7:::
deneme:$y$j9T$POUR...:20716:0:99999:7:::
```

- `$y$` = yescrypt hash. Şifrenin kendisi hiçbir yerde yok.
- `!` = şifre **kilitli**. `ubuntu`'nun şifresi yok, SSH key ile giriyor.
- Sonraki alanlar: son değişiklik günü, min gün, max gün, uyarı.

Neden `sudo` gerekti:

```
$ ls -l /etc/passwd /etc/shadow
-rw-r--r-- 1 root root   1918 /etc/passwd
-rw-r----- 1 root shadow  999 /etc/shadow
```

`passwd` herkese açık (`ls` UID'yi isme çevirmek için okur). `shadow` sadece root ve `shadow` group'una açık.

## 3.3 — Group ile yetki vermek

`deneme` log okuyabilir mi?

```
$ su - deneme -c 'head -3 /var/log/syslog'
cat: /var/log/syslog: Permission denied

$ ls -l /var/log/syslog
-rw-r----- 1 syslog adm 313137 /var/log/syslog
```

`ls -l` formatı:

```
-rw-r-----  1  syslog  adm  313137  Sep 20 22:00  /var/log/syslog
 izinler  link  owner  group  boyut     tarih          isim
```

İzinler 3 blok: **owner / group / others** (`u`/`g`/`o`).
- `rw-` owner (`syslog`): okur, yazar
- `r--` group (`adm`): okur
- `---` others: hiçbir şey

`syslog` bir servis hesabı (`syslog:x:102:102::/nonexistent:/usr/sbin/nologin`). rsyslog bu kimlikle çalışır ve log'u yazar. Okuma yetkisi `adm` group'unda. Yazan ile okuyan ayrılmış.

`deneme` owner değil, `adm`'de değil → others → `---`. Çözüm: group'a ekle.

```
$ sudo usermod -aG adm deneme
$ su - deneme -c 'head -3 /var/log/syslog'
2026-09-20T18:21:48.060807+00:00 ip-172-31-78-215 kernel: Linux version 6.17.0-1017-aws ...
```

Dosyaya dokunmadık, sadece user'ın group üyeliğini değiştirdik.

> ⚠️ `usermod -G` tek başına mevcut ek group'ları **siler**. Her zaman `-aG` (append).

> ⚠️ Group değişikliği açık oturumları etkilemez, çıkıp girmek gerekir.

**`su` notları:**
- `su - deneme` → `deneme` ol. `-` = login shell, onun ortamını sıfırdan kur (home, `$PATH`, `.profile`). Tiresiz kullanma.
- `-c 'komut'` → shell açma, komutu çalıştır çık.
- `su` hedef user'ın şifresini sorar; `sudo` senin şifreni sorar ve sudoers'a bakar.

## 3.4 — Bir programı kim çalıştırabilir?

### Yol 1: Program user yetkisiyle çalışıyorsa → group + `chmod 750`

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
-bash: /usr/local/bin/gizli-program: Permission denied      # ubuntu, gizli'de değil

$ sudo gizli-program
gizli program calisti, ben: root                             # root için izin yok

$ su - deneme -c 'gizli-program'
-bash: line 1: /usr/local/bin/gizli-program: Permission denied

$ sudo usermod -aG gizli deneme
$ su - deneme -c 'gizli-program'
gizli program calisti, ben: deneme
```

Log dosyasıyla aynı mekanizma, `r` yerine `x`.

### Yol 2: Program root gerektiriyorsa → sudoers ile komut bazlı izin

`deneme` root olamıyor:

```
deneme@lev-k:~$ sudo cat /etc/shadow
deneme is not in the sudoers file.
```

Sadece bu komutu root olarak yapabilsin. `visudo` ile (syntax kontrolü yapar, bozuk dosyayı kaydetmez, yoksa `sudo` tamamen kilitlenir):

```
$ sudo visudo -f /etc/sudoers.d/deneme
```

İçine tek satır:

```
deneme  ALL=(root)  /usr/bin/cat /etc/shadow
```

Cümle gibi oku: **KİM, NEREDE = (KİM OLARAK) NE**. `ALL` = her host, format zorunlu kılıyor.

Test:

```
deneme@lev-k:~$ sudo cat /etc/shadow
root:*:20614:0:99999:7:::
...                                                          # çalıştı

deneme@lev-k:~$ sudo cat /etc/passwd
Sorry, user deneme is not allowed to execute '/usr/bin/cat /etc/passwd' as root on lev-k.
```

Aynı `cat`, farklı argüman, red. sudoers komutu **argümanıyla birlikte** eşleştirir.

**sudoers nasıl çalışır:**
- Tek kaynak: `/etc/sudoers`. Son satırı `@includedir /etc/sudoers.d` → o klasördeki dosyaları da aynı listeye ekler. Klasör sadece düzen için.
- Okuyan `sudo` komutunun kendisi, her çalıştığında sıfırdan. Servis/daemon yok.
- `%sudo ALL=(ALL:ALL) ALL` → `%` = group. Ubuntu'daki `sudo` group'u bu satırdan yetki alır.
- Bu makinede `ubuntu`'nun yetkisi aslında cloud-init'in yazdığı dosyadan geliyor:

```
$ sudo cat /etc/sudoers.d/90-cloud-init-users
ubuntu ALL=(ALL) NOPASSWD:ALL
```

`NOPASSWD` → şifre sormaz. Bu yüzden `sudo` hiç şifre istemedi.

### `sudo` vs `su`

| | `sudo` | `su` |
|---|---|---|
| Ne yapar | Tek komutu başkası olarak çalıştır | Başkası olarak shell aç |
| Şifre kimin | Senin | Hedef user'ın |
| İzin nereden | sudoers | Hedefin şifresi |
| Kısıtlanabilir mi | Evet, komut bazlı | Hayır |

## Atlananlar

- **Disk quota** — Ubuntu'da varsayılan kapalı; çok user'lı paylaşımlı sistemler için. Gerektiğinde bakılır.
- `tee` neden `>` yerine — `sudo echo > dosya` çalışmaz çünkü `>`'yi shell senin yetkinle yapar; `tee` bir program olduğu için `sudo` ile root olarak yazar. Ayrıca anlatılacak.

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
