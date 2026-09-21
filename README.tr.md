<div align="right">

[English](README.md) | **Türkçe**

</div>

# Linux Sysadmin Lab

Uygulamalı bir Linux system administration lab'ı: systemd, user & group, apt, logging, process'ler, SSH ve networking — Linux'un gerçekte nasıl çalıştığını okuyarak değil, kurcalayarak öğrenmek.

Tüm deneyler gerçek bir **Ubuntu** makinesinde koşuyor; gösterilen çıktılar gerçek, idealize edilmiş değil.

Bu lab'ın uzun vadeli hedefi virtualization ve Kubernetes. Buradaki hemen her bölüm o dünyanın yapı taşı: kubelet'i systemd çalıştırıyor, user ve permission'lar security context'e dönüşüyor, pod'lar birbiriyle veth pair ve bridge üzerinden konuşuyor. Bu lab, Kubernetes'in altındaki Linux.

## İçindekiler

- [Bölüm 0 — Linux dosya yapısı](#bölüm-0--linux-dosya-yapısı)
- [Bölüm 1 — Everything is a file](#bölüm-1--everything-is-a-file)
- [Bölüm 2 — systemd & systemctl](#bölüm-2--systemd--systemctl)
- [Bölüm 3 — Logging & journalctl](#bölüm-3--logging--journalctl)
- [Bölüm 4 — Users & groups](#bölüm-4--users--groups)
- [Bölüm 5 — Package management (apt)](#bölüm-5--package-management-apt)
- [Bölüm 6 — Process management](#bölüm-6--process-management)
- [Bölüm 7 — SSH & sshd](#bölüm-7--ssh--sshd)
- [Bölüm 8 — Networking temelleri](#bölüm-8--networking-temelleri)
- [Bölüm 9 — File permissions & ownership](#bölüm-9--file-permissions--ownership)
- [Bölüm 10 — Disk & filesystem](#bölüm-10--disk--filesystem)
- [Bölüm 11 — Cron & timers](#bölüm-11--cron--timers)

## Müfredat

| # | Bölüm | Cevapladığı soru | Durum |
|---|-------|------------------|-------|
| 0 | [Linux dosya yapısı](#bölüm-0--linux-dosya-yapısı) | Dizin ağacında ne nerede duruyor — /etc, /var, /usr, /bin ne için var? | ✅ |
| 1 | [Everything is a file](#bölüm-1--everything-is-a-file) | File nedir, file descriptor nedir, socket nedir — ve neden *her şey* bir file? | ⏳ |
| 2 | [systemd & systemctl](#bölüm-2--systemd--systemctl) | systemd makinedeki her programı nasıl kontrol ediyor? | 🔜 |
| 3 | [Logging & journalctl](#bölüm-3--logging--journalctl) | Log'lar nerede yaşıyor, journal nasıl sorgulanır? | 🔜 |
| 4 | [Users & groups](#bölüm-4--users--groups) | User nasıl yaratılır, kısıtlanır, yok edilir — group gerçekte nedir? | ✅ |
| 5 | [Package management (apt)](#bölüm-5--package-management-apt) | `apt install` deyince gerçekte ne oluyor — repo'lar, GPG key'ler, binary'ler? | 🔜 |
| 6 | [Process management](#bölüm-6--process-management) | Process nedir, signal nedir — SIGTERM ile SIGKILL'i gerçekte ne ayırır? | 🔜 |
| 7 | [SSH & sshd](#bölüm-7--ssh--sshd) | sshd nasıl kurulur ve hardening yapılır, key'ler nasıl yönetilir? | 🔜 |
| 8 | [Networking temelleri](#bölüm-8--networking-temelleri) | Makine nasıl konuşuyor — interface'ler, DNS, firewall, namespace'ler? | 🔜 |
| 9 | [File permissions & ownership](#bölüm-9--file-permissions--ownership) | Kim neye dokunabilir — chmod, umask, setuid, ACL? | 🔜 |
| 10 | [Disk & filesystem](#bölüm-10--disk--filesystem) | Disk nasıl directory'ye dönüşüyor — mount, fstab, LVM? | 🔜 |
| 11 | [Cron & timers](#bölüm-11--cron--timers) | Bir işi zamanlayarak nasıl koştururum — cron mu systemd timer mı? | 🔜 |

---

# Bölüm 0 — Linux dosya yapısı

Makine: Ubuntu 24.04 (AWS). Bu bölüm lab pratiği değil, standart FHS (Filesystem Hierarchy Standard) referansı.

## Cheat sheet

| Dizin | Ne saklar | Kalıcı mı | Kim yazar |
|-------|-----------|-----------|-----------|
| `/etc` | sistem geneli config dosyaları (text) | kalıcı | root, paketler kurulumda |
| `/var` | değişen veri: log, cache, spool, db | kalıcı | servisler, root |
| `/usr` | kurulu programlar + kütüphaneler + paylaşılan data | kalıcı, apt'ın yönettiği alan | paket yöneticisi (apt) |
| `/bin` | temel komutlar (`ls`, `cat`...) — Ubuntu'da `/usr/bin`'e symlink | kalıcı | paket yöneticisi |
| `/home` | user'ların kişisel dosyaları | kalıcı | user'ın kendisi |
| `/tmp` | kısa ömürlü geçici dosyalar | periyodik temizlenir (`systemd-tmpfiles`), dağıtıma göre tmpfs ya da disk | herkes (world-writable, sticky bit) |
| `/opt` | apt dışı, kendi kendine yeten 3rd-party yazılım | kalıcı | manuel kurulum |
| `/proc` | çalışan process'ler + kernel durumu — gerçek dosya değil, kernel'in canlı görünümü | RAM'de, disk'te yok | kernel |
| `/sys` | kernel'in device/driver bilgisini export ettiği sanal fs | RAM'de, disk'te yok | kernel |
| `/dev` | device node'ları (`/dev/sda`, `/dev/null`, `/dev/tty1`...) | RAM'de (devtmpfs), disk'te yok | kernel (udev) |
| `/lib` | kernel modülleri + temel programların shared library'leri — Ubuntu'da `/usr/lib`'e symlink | kalıcı | paket yöneticisi |

## 0.1 — `/bin` ve `/lib` neden symlink

Ubuntu "usrmerge" yaptı: eskiden `/bin`, `/sbin`, `/lib` kök dizinde ayrı dururdu, çünkü early boot'ta `/usr` henüz mount edilmemiş olabiliyordu. Artık initramfs her şeyi erken mount ettiği için ayrım anlamsızlaştı; hepsi `/usr` altına taşındı, eski isimler geriye dönük uyumluluk için symlink olarak kaldı.

```
$ ls -la /
lrwxrwxrwx ... bin -> usr/bin
lrwxrwxrwx ... lib -> usr/lib
lrwxrwxrwx ... sbin -> usr/sbin
```

| Yol | Gerçek yeri |
|-----|-------------|
| `/bin/ls` | `/usr/bin/ls` |
| `/sbin/reboot` | `/usr/sbin/reboot` |
| `/lib/systemd` | `/usr/lib/systemd` |

## 0.2 — `/proc`, `/sys`, `/dev`: disk'te yoklar

Üçü de **sanal** dosya sistemi. `df -h` çıktısında görünürler ama disk alanı kullanmazlar; reboot'ta sıfırdan kernel tarafından yaratılırlar.

| Dizin | Ne gösterir | Örnek |
|-------|-------------|-------|
| `/proc/<pid>/` | o process'in durumu | `/proc/1/status`, `/proc/1/cmdline` |
| `/proc/cpuinfo`, `/proc/meminfo` | kernel'in donanım/kaynak görünümü | `cat /proc/meminfo` |
| `/sys/class/net/` | network interface'lerin kernel nesneleri | `/sys/class/net/eth0` |
| `/dev/sda`, `/dev/null`, `/dev/tty1` | device node — bir dosyaya yazmak donanımla konuşmak demektir | `echo hi > /dev/null` |

## 0.3 — `/tmp` vs `/var/tmp` vs `/opt`

| Dizin | Ömür | Kullanım |
|-------|------|----------|
| `/tmp` | kısa — `systemd-tmpfiles` periyodik temizler | kısa ömürlü geçici dosya |
| `/var/tmp` | `/tmp`'den daha uzun tutulur, her zaman disk üzerinde | uzun süren işlerin geçici dosyası |
| `/opt` | kalıcı, apt'ın yönetmediği alan | tek-paket halinde gelen 3rd-party yazılım (`/opt/google/chrome` gibi) |

## Notlar

- `/etc` içindeki her şey neredeyse hep text config'tir, binary olmaz — bu bir kural değil ama yaygın kabul.
- `/usr` altı apt'ın yönettiği alandır: apt install ettiğin her şey buraya düşer, elle dokunulmaz.
- Servis hesaplarının home'u genelde `/nonexistent` ya da `/var/lib/<servis>` olur, `/home` altında değil (bkz. Bölüm 4).

Levent'e sor:
- Bu makinede `/tmp` gerçekten tmpfs (RAM) olarak mı mount edilmiş, yoksa disk üzerinde mi — `mount | grep /tmp` ile doğrulanmadı.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 1 — Everything is a file

> ⏳ Placeholder — file, inode, file descriptor, special file'lar (device, pipe, socket), `lsof`, `/proc/PID/fd`.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 2 — systemd & systemctl

> 🔜 Placeholder — systemd programları nasıl kontrol eder: unit'ler, target'lar, service lifecycle, unit file yazmak.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 3 — Logging & journalctl

> 🔜 Placeholder — journal, unit/zaman/priority ile filtreleme, rsyslog, log rotation.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 4 — Users & groups

Makine: Ubuntu 24.04 (AWS). Ana hesap `ubuntu`, test hesabı `deneme`, servis hesabı `myapp`.

## Cheat sheet

| Komut | Ne yapar |
|---|---|
| `id [user]` | UID, primary GID, ek group'lar |
| `id -gn user` | sadece primary group adı |
| `sudo adduser <user>` | user + group + home + skel kopyası |
| `sudo adduser --system --group --no-create-home <user>` | servis hesabı: UID<1000, nologin, home yok |
| `sudo deluser <user>` / `--remove-home` | user sil / home'u da sil |
| `sudo deluser --system <user>` | servis hesabı sil (flag yoksa reddeder) |
| `sudo groupadd <group>` / `sudo groupdel <group>` | group yarat / sil |
| `sudo usermod -aG <group> <user>` | ek group'a **ekle** (`-a` yoksa listeyi ezer) |
| `sudo gpasswd -a <user> <group>` / `-d <user> <group>` | ek group'a ekle / çıkar |
| `sudo usermod -g <group> <user>` | primary group değiştir (home'daki dosyaları da taşır) |
| `sudo usermod -s <shell> <user>` | login shell değiştir (`/usr/sbin/nologin` = kapat) |
| `sudo usermod -e YYYY-MM-DD <user>` / `-e ''` | hesabı tarihte expire et / kaldır |
| `sudo passwd -l <user>` / `-u <user>` | şifreyi kilitle / aç |
| `sudo chage -l <user>` / `-M 90 <user>` | süre bilgilerini listele / şifre ömrü 90 gün |
| `newgrp <group>` | çıkıp girmeden group'u aktif et (iç shell açar) |
| `su - <user>` / `su - <user> -c 'cmd'` | `<user>` ol / `<user>` olarak tek komut (`<user>`'ın şifresi) |
| `sudo -u <user> cmd` | `<user>` olarak tek komut (senin şifren, shell'i atlar) |
| `sudo visudo -f /etc/sudoers.d/<user>` | sudoers kuralı yaz (syntax kontrollü) |
| `sudo find / -uid <uid>` / `-gid <gid> 2>/dev/null` | öksüz dosyaları bul |
| `sudo chgrp <group> dosya` | dosyanın group'unu değiştir |
| `cut -d: -f1 /etc/passwd` | tüm user adları |

## 4.1 — Kimlik

```
$ id
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),24(cdrom),27(sudo),30(dip),105(lxd)
```

| Alan | Anlamı |
|---|---|
| `uid` | Linux seni isimle değil bu sayıyla tanır. 0 root, 1–999 sistem, 1000+ insan |
| `gid` | primary group. Yarattığın dosyalar buna ait olur. Ubuntu her user'a kendi adıyla tek kişilik group açar |
| `groups` | ek group'lar. Yetkiler çoğunlukla buradan gelir |

| Group | Verdiği |
|---|---|
| `sudo` | root olabilirsin (`/etc/sudoers`'taki `%sudo` satırı) |
| `adm` | `/var/log` okuyabilirsin, başka hiçbir şey |
| `lxd` | LXD container yönetimi (Docker değil, Canonical'ın aracı). Pratikte root'a eşdeğer |
| `cdrom`, `dip` | eski donanım group'ları, anlamsız |

User + group birlikte çalışır: owner'san user'ın, değilsen group üyeliğin belirler. Yönetim alışkanlığı group üzerinden ("5 kişi log okusun" = 5 user'ı `adm`'e ekle).

## 4.2 — Dört dosya

User/group sisteminin tamamı 4 metin dosyası. Veritabanı yok, servis yok. `adduser`, `usermod`, `gpasswd` bunları düzenleyen programlar.

| Dosya | Ne verir | Kim okur |
|---|---|---|
| `/etc/passwd` | user kimliği: isim, UID, primary GID, home, shell | herkes |
| `/etc/shadow` | user şifre hash'i + süre kuralları | root, `shadow` group |
| `/etc/group` | group kimliği: isim, GID, **ek** üyeler | herkes |
| `/etc/gshadow` | group şifresi (pratikte hep boş) | root |

Kimlik açık çünkü `ls -l` UID→isim çevirmek zorunda. Hash kapalı çünkü offline kırılabilir.

**`/etc/passwd`**

```
deneme : x : 1001 : 1001 : Jack Brown,31,n/a,n/a,n/a : /home/deneme : /bin/bash
 isim   şifre UID   GID           GECOS                  home         shell
```

`x` = şifre shadow'da. Shell `nologin` ise login kapalı ama hesap var.

**`/etc/shadow`**

```
deneme : $y$j9T$POUR... : 20716 : 0 : 99999 : 7 : : :
 isim        hash         son    min   max  uyarı inactive expire
```

| Değer | Anlamı |
|---|---|
| `$y$` | yescrypt hash. Şifrenin kendisi hiçbir yerde yok |
| `!` veya `*` başta | kilitli. `ubuntu:!:...` → şifresi yok, SSH key ile giriyor |
| `20716` | 1970'ten beri gün sayısı |
| `99999` | asla |

**`/etc/group`**

```
adm : x : 4 : syslog,ubuntu,deneme
isim şifre GID     ek üyeler
```

Primary üyeler burada görünmez, `passwd`'deki GID'de yazılı. `deneme:x:1001:` boş = kimse ek üye değil, ama `deneme` primary olarak içinde.

## 4.3 — User yaratma

```
$ sudo adduser deneme
info: Adding new group `deneme' (1001) ...
info: Adding new user `deneme' (1001) with group `deneme (1001)' ...
info: Creating home directory `/home/deneme' ...
info: Copying files from `/etc/skel' ...
New password:
```

| Adım | Nereye |
|---|---|
| user satırı | `/etc/passwd` |
| hash | `/etc/shadow` |
| tek kişilik group | `/etc/group` |
| home | `/home/deneme`, `drwxr-x---` (others giremez) |
| şablon | `/etc/skel` → home'a kopya, sahiplik user'a |

`/etc/skel` = iskelet: `.bashrc`, `.profile`, `.bash_logout`. Buraya konan her şey sonraki her user'a gider.

## 4.4 — `ls -l` formatı

```
-rw-r-----  1  syslog  adm  313512  Sep 20 22:00  /var/log/syslog
 izinler  link  owner  group  boyut     tarih          isim
```

İzinler 3 blok: **owner / group / others** (`u`/`g`/`o`). İlk karakter tip: `-` dosya, `d` dizin, `l` symlink.

| Blok | `syslog` için | Kim |
|---|---|---|
| `rw-` | okur, yazar | owner `syslog` (rsyslog bu kimlikle yazar) |
| `r--` | okur | group `adm` |
| `---` | hiçbir şey | others |

`syslog` bir servis hesabı: `syslog:x:102:102::/nonexistent:/usr/sbin/nologin`. Yazan ile okuyan ayrılmış.

## 4.5 — Group ile yetki

`deneme` `adm`'de değil → others → `---`:

```
$ su - deneme -c 'head -3 /var/log/syslog'
cat: /var/log/syslog: Permission denied
```

Dosyaya dokunmadan, user'ı group'a ekle:

```
$ sudo usermod -aG adm deneme
$ su - deneme -c 'head -3 /var/log/syslog'
2026-09-20T18:21:48.060807+00:00 ip-172-31-78-215 kernel: Linux version 6.17.0-1017-aws ...
```

| İş | `usermod` | `gpasswd` |
|---|---|---|
| ekle | `sudo usermod -aG adm deneme` | `sudo gpasswd -a deneme adm` |
| çıkar | `sudo usermod -G users,gizli deneme` (tüm listeyi yeniden yaz) | `sudo gpasswd -d deneme adm` |
| sıra | flag → group → user | flag → user → group |
| tehlike | `-a` unutulursa liste ezilir | yok |

⚠️ Group değişikliği açık oturumu etkilemez; çıkıp gir veya `newgrp` (4.9).

**`su` vs `sudo`**

| | `sudo` | `su` |
|---|---|---|
| ne yapar | tek komutu başkası olarak çalıştır | başkası olarak shell aç |
| şifre | senin | hedefin |
| izin | sudoers | hedefin şifresini bilmek |
| kısıtlanır mı | evet, komut bazlı | hayır |
| hedef `nologin` ise | çalışır (shell'i atlar) | çalışmaz |

`su - <user>`: `-` = login shell, <user>'in ortamını sıfırdan kur. Tiresiz kullanma. `-c 'cmd'` = shell açma, komutu çalıştır çık.

## 4.6 — Bir programı kim çalıştırır

**Yol 1: program user yetkisiyle çalışıyor → dosya group'u + `chmod 750`**

```
$ sudo groupadd gizli
$ printf '#!/bin/bash\necho "gizli program calisti, ben: $(whoami)"\n' | sudo tee /usr/local/bin/gizli-program
$ sudo chmod 750 /usr/local/bin/gizli-program
$ sudo chown root:gizli /usr/local/bin/gizli-program
$ ls -l /usr/local/bin/gizli-program
-rwxr-x--- 1 root gizli 57 /usr/local/bin/gizli-program
```

| Kim | Sonuç | Neden |
|---|---|---|
| `ubuntu` | denied | `gizli`'de değil |
| `sudo` | `ben: root` | izinler root'a işlemez |
| `deneme` | denied | `gizli`'de değil |
| `deneme` (`usermod -aG gizli` sonrası) | `ben: deneme` | group'a girdi |

Log dosyasıyla aynı mekanizma, `r` yerine `x`.

**Yol 2: program root gerektiriyor → sudoers'ta komut bazlı kural**

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

Cümle gibi oku: **KİM, NEREDE = (KİM OLARAK) NE**. `ALL` = her host, format zorunlu kılıyor.

```
deneme@lev-k:~$ sudo cat /etc/shadow
root:*:20614:0:99999:7:::                    # çalıştı
deneme@lev-k:~$ sudo cat /etc/passwd
Sorry, user deneme is not allowed to execute '/usr/bin/cat /etc/passwd' as root on lev-k.
```

Aynı `cat`, farklı argüman, red. sudoers komutu **argümanıyla** eşleştirir.

| sudoers | Not |
|---|---|
| `/etc/sudoers` | ana dosya, son satırı `@includedir /etc/sudoers.d` |
| `/etc/sudoers.d/*` | ekler, aynı listeye yapıştırılır. Düzen için |
| `visudo` | kilitler, syntax kontrol eder, bozuk kaydetmez. Bozuk sudoers = `sudo` kilitlenir |
| `%sudo ALL=(ALL:ALL) ALL` | `%` = group. Ubuntu `sudo` group'u buradan |
| `ubuntu ALL=(ALL) NOPASSWD:ALL` | cloud-init'in yazdığı, `/etc/sudoers.d/90-cloud-init-users`. Şifre sormama sebebi |
| okuyan | `sudo` komutunun kendisi, her seferinde. Daemon yok |

## 4.7 — Group silme ve öksüz GID

```
$ sudo gpasswd -d deneme adm
Removing user deneme from group adm
$ sudo groupdel gizli                          # ek üyeler otomatik düşer, primary ise reddeder
$ ls -l /usr/local/bin/gizli-program
-rwxr-x--- 1 root 1002 57 /usr/local/bin/gizli-program
```

Group gitti, dosyada GID 1002 kaldı, `ls` isme çeviremiyor. Tehlike: 1002'yi alan yeni group bu dosyayı miras alır.

```
$ sudo find / -gid 1002 2>/dev/null
/usr/local/bin/gizli-program
$ sudo chgrp root /usr/local/bin/gizli-program
```

`2>/dev/null` = stderr'i yut (`find` `/proc`'ta kendi kuyruğunu kovalar, gürültü). Doğru sıra: **önce `find`, sonra `groupdel`/`deluser`**.

## 4.8 — Primary group

Tek işi: yarattığın yeni dosyanın group'u. Ek group "nereye erişirim", primary "yarattığım kime ait".

```
$ sudo usermod -g gizli deneme                 # küçük -g = primary
$ grep '^deneme:' /etc/passwd
deneme:x:1001:1002:...                         # GID 1001 → 1002
```

⚠️ `usermod -g` **home'daki** eski primary'ye ait dosyaları da yeni group'a taşır. Home dışına dokunmaz. Geri: `sudo usermod -g deneme deneme`.

Umask notu: primary group adı = user adı ise umask `002` (`-rw-rw-r--`), değilse `022` (`-rw-r--r--`). Bölüm 9.

## 4.9 — `newgrp`: çıkıp girmeden group

Shell group listesini **login anında** kopyalar. `usermod` dosyayı değiştirir, açık shell'i değil:

```
deneme@lev-k:~$ cat /etc/group | grep gizli
gizli:x:1002:deneme                            # dosyada üye
deneme@lev-k:~$ id
uid=1001(deneme) gid=1001(deneme) groups=1001(deneme),100(users)   # shell'de yok
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
| ne yapar | iç shell açar (`$SHLVL` 1→2), `<group>`'ı ekler **ve primary yapar** |
| şart | `/etc/group`'ta üye olmak; değilsen group şifresi (yok) → red |
| kalıcı mı | hayır, `exit` ile eski shell'e dön |
| `sg <group> -c 'cmd'` | shell açmadan tek komut |

```
deneme@lev-k:~$ touch test1.txt               # iç shell'de → group gizli
deneme@lev-k:~$ exit
deneme@lev-k:~$ touch test2.txt               # dış shell'de → group deneme
-rw-rw-r-- 1 deneme gizli  0 test1.txt
-rw-rw-r-- 1 deneme deneme 0 test2.txt
```

Process ağacı:
```
su (root) → -bash (deneme, login) → newgrp → bash (deneme, gizli aktif)
```

## 4.10 — User kısıtlama

Silmeden erişimi kesmek. Hepsi `passwd`/`shadow`'daki bir alanı değiştirir.

| Komut | Kapatır | Açık kalır | Senaryo |
|---|---|---|---|
| `passwd -l <user>` | şifreyle login (hash başına `!`) | SSH key, cron, çalışan process | izin, geçici askı |
| `usermod -s /usr/sbin/nologin <user>` | her türlü interaktif login | cron, çalışan process | kalıcı kapatma, servis hesabı |
| `usermod -e 2026-12-31 <user>` | tarihten sonra her şey | tarihe kadar her şey | stajyer, geçici erişim |
| `chage -M 90 <user>` | 90 gün sonra şifre zorla değişir | her şey | şifre politikası |

| Geri alma | |
|---|---|
| `passwd -u <user>` | kilidi aç, eski şifre çalışır |
| `usermod -s /bin/bash <user>` | shell'i geri ver |
| `usermod -e '' <user>` | expire'ı kaldır |
| `chage -l <user>` | tüm süre bilgilerini okunabilir göster |

Hata mesajları farklı, nerede takıldığını söyler:

```
$ su - deneme                                  # passwd -l sonrası
su: Authentication failure                     # şifre aşaması
$ su - deneme                                  # nologin sonrası
This account is currently not available.       # shell aşaması
$ su - deneme                                  # usermod -e 2026-01-01 sonrası
Your account has expired; please contact your system administrator.
```

Shell alanına herhangi bir program konabilir (kiosk menüsü, `git-shell`, `rrsync`):

```
$ printf '#!/bin/bash\necho "Buraya giris yok canim :)"\n' | sudo tee /usr/local/bin/giris-yok
$ sudo chmod 755 /usr/local/bin/giris-yok
$ sudo usermod -s /usr/local/bin/giris-yok deneme
$ su - deneme
Password:
Buraya giris yok canim :)
```

## 4.11 — Servis hesabı

Uygulamayı root olarak çalıştırma; hack'lenirse saldırgan root olur. Uygulamaya özel, login yapamayan, home'suz user. `syslog`, `sshd`, `www-data` böyle. Kubernetes `runAsUser` aynı fikir.

```
$ sudo adduser --system --group --no-create-home myapp
$ grep -E '^(syslog|myapp):' /etc/passwd
syslog:x:102:102::/nonexistent:/usr/sbin/nologin
myapp:x:111:113::/nonexistent:/usr/sbin/nologin
```

| Flag | Ne yapar |
|---|---|
| `--system` | UID 100–999, shell `nologin`, şifre ve GECOS sorma |
| `--group` | aynı isimde group, primary yap (yoksa `nogroup`) |
| `--no-create-home` | `/home/myapp` açma; çalışma dizini `/var/lib/myapp` olur |
| `--uid 5000` | aralığı ezer; "1000 altı = servis" geleneğini bozar, gerekmedikçe kullanma |

UID aralığı gelenek, kernel sınırı değil (OpenShift 1000000000+ kullanır). Kullanmak: login yok, şifre yok, `su` çalışmaz → `sudo -u myapp cmd` veya systemd `User=myapp`.

```
$ sudo -u myapp whoami
myapp
$ sudo -u myapp gizli-program
sudo: unable to execute /usr/local/bin/gizli-program: Permission denied    # sıfır yetki, doğru başlangıç
```

### Case study: `myapp` servisi

Hedef: script her saniye `/var/lib/myapp/log.txt`'ye tarih yazar, systemd `myapp` olarak çalıştırır, herkes okur, kimse yazamaz, script'i sadece `myapp` çalıştırır.

| Adım | Komut |
|---|---|
| hesap | `sudo adduser --system --group --no-create-home myapp` |
| dizin | `sudo mkdir /var/lib/myapp && sudo chown myapp:myapp /var/lib/myapp && sudo chmod 755 /var/lib/myapp` |
| script | `sudo nano /usr/local/bin/myapp-logger` |
| sahiplik | `sudo chown myapp:myapp /usr/local/bin/myapp-logger && sudo chmod 700 /usr/local/bin/myapp-logger` |
| elle test | `sudo -u myapp timeout 25 /usr/local/bin/myapp-logger` |
| unit | `sudo nano /etc/systemd/system/myapp.service` |
| başlat | `sudo systemctl daemon-reload && sudo systemctl start myapp` |
| kontrol | `systemctl status myapp`, `ps -o user,pid,cmd -C myapp-logger` |
| izle | `tail -f /var/lib/myapp/log.txt` |

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

| systemd | Anlamı |
|---|---|
| `User=myapp` | servis bu kimlikle çalışır |
| `Restart=always` | çökerse yeniden başlat; `systemctl stop` buna girmez |
| `[Install] WantedBy=` | `enable` deyince açılışta ne zaman başlasın |
| `daemon-reload` | unit dosyalarını yeniden oku; çalışan servise dokunmaz. Unit'e her dokunuşta |
| `start` / `stop` / `restart` | şimdi başlat / durdur / durdur+başlat |
| `enable` / `enable --now` | açılışta başlasın / + şimdi de başlat |

Süreç:

1. `chmod 700` + `chown myapp` → `ubuntu` çalıştıramaz: `Permission denied`. Root çalıştırır, izinler ona işlemez.
2. Elle test: `timeout 25` sonsuz döngüyü 25 sn sonra öldürür, terminal kilitlenmez.
3. `deneme` okur, yazamaz:
   ```
   deneme@lev-k:~$ tail -f /var/lib/myapp/log.txt
   Mon Sep 21 12:43:47 UTC 2026
   deneme@lev-k:~$ echo hack >> /var/lib/myapp/log.txt
   -bash: /var/lib/myapp/log.txt: Permission denied
   ```
4. Servis `myapp` olarak çalışıyor:
   ```
   $ ps -o user,pid,cmd -C myapp-logger
   USER         PID CMD
   myapp       3395 /bin/bash /usr/local/bin/myapp-logger
   ```
5. `systemctl stop` `sudo`suz → polkit `ubuntu`'nun şifresini ister, yok, red. `sudo` ile sudoers'a bakar, geçer.

Temizlik: `stop` → `rm unit` → `daemon-reload` → `deluser --system myapp` → `rm -rf /var/lib/myapp` → `rm script`.

## 4.12 — User silme

| Komut | Home | Diğer dosyalar |
|---|---|---|
| `sudo deluser <user>` | kalır | dokunmaz |
| `sudo deluser --remove-home <user>` | silinir | dokunmaz |

Diğer yerlerdeki dosyalar UID ile öksüz kalır; aynı UID'yi alan yeni user miras alır.

```
$ sudo -u deneme touch /tmp/deneme-dosyasi
$ sudo deluser --remove-home deneme
userdel: user deneme is currently used by process 3793
```

Açık process'i olan user silinmez. `ps -o user,pid,cmd -p 3793` → `deneme -bash`. İnteraktif bash SIGTERM'i yok sayar (işini kaybetmesin diye), `kill -9` gerekir:

```
$ sudo kill -9 3793
$ sudo deluser --remove-home deneme
$ ls -l /tmp/deneme-dosyasi
-rw-rw-r-- 1 1001 1001 0 /tmp/deneme-dosyasi       # isim yok, sadece UID
$ sudo find / -uid 1001 2>/dev/null
/tmp/deneme-dosyasi
```

Doğru sıra: **`find -uid` → sil/`chown` → `deluser`**.

## Notlar

**`usermod` flag'leri** (user modify, Ubuntu'da `passwd` paketinden):

| Flag | İngilizce | Alan |
|---|---|---|
| `-s` | shell | passwd 7 |
| `-g` | group | passwd 4 (primary) |
| `-aG` | append groups | `/etc/group` |
| `-e` | expire | shadow 8 |
| `-d` | directory | passwd 6 |
| `-l` | login | passwd 1 (yeniden adlandır) |
| `-L` / `-U` | lock / unlock | = `passwd -l/-u` |

**`sudo`'nun `>` sorunu:** `sudo echo x > /root/f` çalışmaz; `>`'yi shell senin yetkinle yapar, `sudo` sadece `echo`'ya işler. `tee` bir program, `sudo` ile root olarak yazar: `echo x | sudo tee /root/f`. Aynı sebeple `sudo rm /home/deneme/test*.txt` çalışmaz (`*`'ı senin shell'in açar, dizine giremez): `sudo sh -c 'rm /home/deneme/test*.txt'`.

**`/run/sudo/ts/UID`:** `sudo`'nun 15 dk şifre sormama hafızası. `sudo -k` sıfırlar.

**Atlanan:** disk quota (Ubuntu'da kapalı, çok user'lı paylaşımlı sistemler için).

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 5 — Package management (apt)

> 🔜 Placeholder — apt nasıl çalışır: repository'ler, sources list, GPG key'ler, binary kurulumu, update.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 6 — Process management

> 🔜 Placeholder — ps, top/htop, signal'lar (SIGTERM vs SIGKILL), nice/renice.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 7 — SSH & sshd

> 🔜 Placeholder — sshd kurulumu, key management, sshd_config, hardening.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 8 — Networking temelleri

> 🔜 Placeholder — ip, ss, ping, DNS, firewall (ufw → nftables), network namespace'ler.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 9 — File permissions & ownership

> 🔜 Placeholder — chmod, chown, umask, setuid/setgid, ACL.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 10 — Disk & filesystem

> 🔜 Placeholder — mount, fstab, lsblk, df/du, LVM'e giriş.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 11 — Cron & timers

> 🔜 Placeholder — cron vs systemd timer.

[↑ İçindekilere dön](#i̇çindekiler)
