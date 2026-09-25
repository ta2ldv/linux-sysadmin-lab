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
| 2 | [systemd & systemctl](#bölüm-2--systemd--systemctl) | systemd makinedeki her programı nasıl kontrol ediyor? | ✅ |
| 3 | [Logging & journalctl](#bölüm-3--logging--journalctl) | Log'lar nerede yaşıyor, journal nasıl sorgulanır? | ✅ |
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

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 1 — Everything is a file

> ⏳ Placeholder — file, inode, file descriptor, special file'lar (device, pipe, socket), `lsof`, `/proc/PID/fd`.

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 2 — systemd & systemctl

Makine: Ubuntu 24.04 (AWS), hesap `ubuntu` (sudo).

## Cheat sheet

| Komut | Ne yapar |
|---|---|
| `ps -p 1 -o pid,comm` | PID 1'in kim olduğunu göster (systemd) |
| `ps -ef --forest` | process ağacını girintili çiz, parent/child ilişkisini göster |
| `ps -p <pid> -o pid,ppid` | bir process'in parent PID'ini göster |
| `systemctl status <servis>` | durumu göster |
| `sudo systemctl start <servis>` | şimdi başlat |
| `sudo systemctl stop <servis>` | şimdi durdur |
| `sudo systemctl restart <servis>` | durdur + tekrar başlat |
| `sudo systemctl reload <servis>` | config'i process'i öldürmeden yeniden oku (destekleyen servislerde) |
| `sudo systemctl enable <servis>` | boot'ta otomatik başlasın (symlink oluşturur) |
| `sudo systemctl disable <servis>` | boot'ta otomatik başlamasın (symlink siler) |
| `journalctl -u <servis> -n N` | son N log satırı |
| `journalctl -u <servis> -f` | log'u canlı takip et |
| `ls -la /etc/systemd/system/` | admin'in elle eklediği/enable ettiği unit'ler ve symlink'ler |

## 2.1 — PID 1: systemd

Kernel boot sırasında ilk çalıştırdığı process systemd; sistemdeki her diğer process (servisler, shell'in, SSH bağlantın) doğrudan ya da dolaylı olarak onun child'ı.

```
$ ps -p 1 -o pid,comm
    PID COMMAND
      1 systemd
```

| Flag | Anlamı |
|---|---|
| `-p` | PID'e göre filtrele |
| `-o pid,comm` | sadece PID ve process ismini (command) göster |

Pratik önemi: `systemctl` komutları çalışıyor çünkü systemd'ye konuşuyorsun; `kill -9 1` teorik olarak sistemi çökertir çünkü kökü öldürmüş olursun.

## 2.2 — systemd bir daemon, systemctl bir client

`systemd` (PID 1) arka planda sürekli çalışan bir daemon — servisleri başlatma/durdurma, unit dosyalarını okuma gibi asıl işi o yapar. `systemctl` ise kendisi işi yapmaz, systemd'ye "şunu yap" diye mesaj gönderir.

Bu iletişim **D-Bus** üzerinden olur — Linux'ta process'lerin birbiriyle konuşması için kullanılan bir IPC (inter-process communication) sistemi. `systemctl` bir D-Bus mesajı gönderir, systemd (D-Bus üzerinde dinleyen taraf) bu mesajı alıp işler. Sadece systemd değil, NetworkManager, bluetooth stack'i gibi birçok Linux servisi de D-Bus kullanır.

## 2.3 — Process ağacı: parent/child

`ps -ef --forest` çıktıyı ağaç gibi çizer — girinti ve `└─` karakterleriyle hangi process'in hangisinin child'ı olduğunu gösterir. En üstte systemd (PID 1), altına doğru dallanarak diğer her şey.

| Komut | Ne için |
|---|---|
| `ps -ef --forest \| grep sshd` | sadece sshd dalını filtrele |
| `ps -ef --forest \| less` sonra `/sshd` | interaktif arama; `n` sonraki eşleşme, `q` çıkış |

```
systemd (PID 1)
 └─ sshd listener (857, root)               — sürekli çalışan, bağlantı bekleyen ana sshd
     ├─ sshd: ubuntu [priv] (858)            — pts/0 için privilege-separation process'i
     │   └─ sshd: ubuntu@pts/0 (1003)        — ilk SSH oturumu
     │       └─ -bash (1049)
     └─ sshd: ubuntu [priv] (860)            — pts/1 (şu an kullanılan)
         └─ sshd: ubuntu@pts/1 (1048)
             └─ -bash (1058)
                 ├─ ps -ef --forest (1144)
                 └─ less (1145)
```

`ps -p <pid> -o pid,ppid` ile zinciri tek tek doğrulamak:

| PID | PPID |
|---|---|
| 1058 | 1048 |
| 1048 | 860 |
| 860 | 857 |
| 857 | 1 |
| 1 | 0 |

PID 1'in PPID'i 0 — yani kernel'in kendisi.

## 2.4 — systemd unit dizinleri

| Dizin | Öncelik | Kim yazar |
|---|---|---|
| `/etc/systemd/system/` | yüksek | sysadmin (elle eklenen/override edilen unit'ler, `enable` ile oluşan symlink'ler) |
| `/run/systemd/system/` | orta | systemd/programlar (runtime'da oluşan geçici unit'ler, reboot'ta silinir) |
| `/usr/lib/systemd/system/` (symlink: `/lib/systemd/system/`) | düşük | apt/dpkg (paketlerin kurduğu default unit dosyaları) |

| Config/state dosyaları | Ne saklar |
|---|---|
| `/etc/systemd/*.conf` (`system.conf`, `journald.conf`, `logind.conf`...) | systemd'nin kendi davranış ayarları |
| `/var/lib/systemd/` | systemd'nin durum/state dosyaları |
| `/var/log/journal/` | journalctl loglarının durduğu yer (kalıcıysa) |

## 2.5 — Dizin içini incelemek: symlink'ler

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

| Girdi tipi | Örnek | Ne demek |
|---|---|---|
| symlink (`l` ile başlar) | `chronyd.service -> .../chrony.service` | biri `systemctl enable chronyd` demiş; `enable` dosyayı kopyalamaz, `/usr/lib/systemd/system/`'deki orijinale işaret eden bir symlink oluşturur |
| `.wants`/`.requires` dizini | `multi-user.target.wants/` | "bu target çalıştığında şunlar da çalışsın" listesi — içi hep sadece symlink |
| düz dosya (`-` ile başlar) | `snap-core22-2411.mount` | gerçek unit dosyası, genelde otomatik üretilmiş (snap paketleri) |

Doğrulama:

```
$ ls -l /etc/systemd/system/chronyd.service
lrwxrwxrwx 1 root root 38 Jun 10 10:16 /etc/systemd/system/chronyd.service -> /usr/lib/systemd/system/chrony.service
$ ls -l /usr/lib/systemd/system/chrony.service
-rw-r--r-- 1 root root 1923 Jul  2  2024 /usr/lib/systemd/system/chrony.service
```

## 2.6 — `.wants` mekanizması ve target'lar

`multi-user.target` = sistemin "network açık, çoklu kullanıcı çalışabilir, grafik arayüz yok" durumuna ulaştığı, sunucularda kullanılan normal boot hedefi.

| Kural | Açıklama |
|---|---|
| `.wants`/`.requires` içi | kesinlikle sadece symlink, hiç gerçek dosya olmaz |
| Hangi target'a bağlı | unit dosyasının `[Install]` bölümündeki `WantedBy=<target>` satırına göre değişir; `enable` symlink'i o target'ın `.wants/` dizinine koyar |
| Boot ile ilişkisi | systemd boot'ta bir default target'a (genelde `multi-user.target`) ulaşmaya çalışır, bu sırada o target'ın `.wants/` dizinindeki tüm servisleri başlatır |

## 2.7 — Unit dosya yapısı

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

| Bölüm | İçerik |
|---|---|
| `[Unit]` | kimlik ve bağımlılıklar — `Description=`, `After=`, `Before=`, `Requires=`, `Wants=` |
| `[Service]` | tipe özel — `ExecStart=`, `ExecStop=`, `Restart=`, `Type=`, `User=` |
| `[Install]` | enable edilince ne olsun, hangi target'a bağlansın — `WantedBy=`, `RequiredBy=`, `Alias=` |

| Unit tipi | Ne temsil eder | Örnek |
|---|---|---|
| `.service` | bir process/daemon | `cron.service`, `nginx.service` |
| `.mount` | bir dosya sistemi mount noktası | `boot-efi.mount` |
| `.socket` | bir network/IPC socket — bağlantı gelince ilişkili servisi tetikler | `docker.socket` |
| `.timer` | zamanlanmış görev, cron'a benzer | `apt-daily.timer` |
| `.target` | bir grup unit'i temsil eden senkronizasyon noktası (gerçek process değil) | `multi-user.target` |

### Case study: `logger-demo` servisi yazmak

| Adım | Komut |
|---|---|
| script yaz | `sudo tee /usr/local/bin/logger-demo.sh > /dev/null << 'EOF' ... EOF` + `sudo chmod +x` |
| unit file yaz | `sudo tee /etc/systemd/system/logger-demo.service > /dev/null << 'EOF' ... EOF` |
| yüklendi mi kontrol | `systemctl status logger-demo` |
| başlat | `sudo systemctl start logger-demo` |
| durum kontrol | `systemctl status logger-demo` |
| boot'ta başlasın | `sudo systemctl enable logger-demo` |
| logları izle | `journalctl -u logger-demo -n 5` / `-f` |
| durdur + devre dışı bırak | `sudo systemctl stop logger-demo` / `sudo systemctl disable logger-demo` |
| reboot testi (disabled) | `sudo reboot` → `systemctl status logger-demo` |
| reboot testi (enabled) | `sudo systemctl enable logger-demo` → `sudo reboot` → `systemctl status logger-demo` |

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

Süreç:

1. Unit dosyasını `/etc/systemd/system/`'e yazar yazmaz `systemctl status logger-demo` onu zaten tanıyor — ama `start` etmediğin için `inactive`:
   ```
   ○ logger-demo.service - Demo heartbeat logger
        Loaded: loaded (/etc/systemd/system/logger-demo.service; disabled; preset: enabled)
        Active: inactive (dead)
   ```
2. `sudo systemctl start logger-demo` sonrası çalışıyor, ama `Loaded` hâlâ `disabled` — **start ve enable birbirinden bağımsız iki adım**, start etmek boot'ta otomatik başlayacağı anlamına gelmiyor:
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
   `ls /etc/systemd/system/multi-user.target.wants/ | grep logger` bu noktada **boş** döner — henüz `enable` edilmedi.
3. `sudo systemctl enable logger-demo` iki şey yapar: `Loaded` karşısında `enabled` yazar, ve `.wants/` dizininde symlink oluşturur:
   ```
   $ sudo systemctl enable logger-demo
   Created symlink /etc/systemd/system/multi-user.target.wants/logger-demo.service → /etc/systemd/system/logger-demo.service.
   $ ls -la /etc/systemd/system/multi-user.target.wants/logger-demo.service
   lrwxrwxrwx 1 root root 39 Sep 21 22:03 ... -> /etc/systemd/system/logger-demo.service
   ```
4. `journalctl -u logger-demo -n 5` servisin loglarını gösterir; `-f` canlı takip (`^C` ile çık):
   ```
   Sep 21 22:05:05 lev-k logger-demo.sh[1901]: 2026-09-21 22:05:05 - heartbeat #60
   Sep 21 22:05:10 lev-k logger-demo.sh[1901]: 2026-09-21 22:05:10 - heartbeat #61
   ```
5. `stop` process'i `SIGTERM` ile öldürür (`code=killed, signal=TERM`), `disable` symlink'i siler:
   ```
   $ sudo systemctl disable logger-demo
   Removed "/etc/systemd/system/multi-user.target.wants/logger-demo.service".
   ```
6. **Reboot test 1 (disabled):** `sudo reboot` öncesi `Loaded: disabled` / `Active: inactive`. Reboot sonrası aynı: hâlâ `disabled` / `inactive` — disabled bir servis reboot'ta kendiliğinden başlamıyor.
7. **Reboot test 2 (enabled):** `sudo systemctl enable logger-demo` sonrası `.wants/` symlink'i tekrar oluşur, `Loaded: enabled` ama henüz `Active: inactive` (başlatılmadı). `sudo reboot` sonrası servis **yeni bir PID ile** otomatik çalışıyor:
   ```
   ● logger-demo.service - Demo heartbeat logger
        Loaded: loaded (/etc/systemd/system/logger-demo.service; enabled; preset: enabled)
        Active: active (running) since Mon 2026-09-21 22:15:36 UTC; 24s ago
      Main PID: 525 (logger-demo.sh)
   Sep 21 22:15:36 lev-k systemd[1]: Started logger-demo.service - Demo heartbeat logger.
   ```
   Process state reboot'ta hiç korunmaz (eski PID 1901 gitti, yeni PID 525); korunan tek şey `.wants/` dizinindeki symlink — yani "bu servis boot'ta başlasın" bilgisi.

## Kubernetes ile ilişkisi

- Bir Kubernetes node'unda `kubelet`, host üzerinde systemd tarafından yönetilen sıradan bir servis; `systemctl status kubelet` / `journalctl -u kubelet -f` bu bölümde öğrenilenle birebir aynı.
- Container runtime (`containerd` ya da `CRI-O`) da ayrı bir systemd unit'i olarak çalışır — kubelet onunla CRI (Container Runtime Interface) üzerinden konuşur, systemd ikisini de ayrı ayrı ayakta tutar.
- **cgroup driver**: hem kubelet hem runtime, container'ların resource limit'lerini (cgroup) yönetmek için ya `systemd` driver'ını ya da eski `cgroupfs` driver'ını kullanır. İkisi farklı driver kullanırsa aynı makinede iki ayrı cgroup yönetim otoritesi oluşur, node kararsızlaşabilir.
- Bu yüzden kubelet ve runtime'ın **aynı** cgroup driver'ında olması zorunlu; systemd tabanlı dağıtımlarda (Ubuntu dahil) önerilen `systemd` driver'ıdır, çünkü systemd zaten sistemin tek cgroup manager'ı olarak çalışıyor.
- OpenShift/RHEL CoreOS tarafında da aynı model geçerli: kubelet ve CRI-O yine systemd unit'leri olarak çalışır, node config'i (machine-config-operator) da bu unit dosyalarını günceller.

## Notlar

| Flag | Anlamı |
|---|---|
| `ps -p` | PID'e göre filtrele |
| `ps -o pid,comm` / `pid,ppid` | sadece istenen kolonları göster |
| `ps -ef --forest` | ağaç görünümü |

- `start` ve `enable` bağımsız: `start` şimdi çalıştırır, `enable` boot'ta çalıştırır. Birini yapmak diğerini yapmaz.
- `.wants`/`.requires` dizinlerinin içi hep sadece symlink; unit dosyasının kendisi `/etc/systemd/system/` ya da `/usr/lib/systemd/system/`'de durur.
- Reboot test'i process state'in korunmadığını gösterdi (yeni PID); korunan tek şey enable durumu (symlink var mı yok mu).

[↑ İçindekilere dön](#i̇çindekiler)

---

# Bölüm 3 — Logging & journalctl

Makine: Ubuntu 24.04 (AWS), hesap `ubuntu` (`adm` group'unda — journalctl/`/var/log` okuma yetkisi buradan gelir).

## Cheat sheet

| Komut | Ne yapar |
|---|---|
| `logger "mesaj"` | test mesajı gönder (klasik `syslog()` çağrısı, `/dev/log` üzerinden) |
| `logger -p crit "mesaj"` | belirli priority ile gönder (`crit`+ mesajlar journald'da anında fsync tetikler) |
| `logger -u <socket> "mesaj"` | `/dev/log` yerine doğrudan bir Unix socket'e yaz |
| `journalctl -u <unit>` | belirli unit'in loglarını göster |
| `journalctl -u <unit> -n N` | son N satır |
| `journalctl --since ... --until ...` | zaman aralığına göre filtrele |
| `journalctl -p <sev>` / `-p a..b` | severity'ye göre filtrele (tek değer veya aralık, ikisi de dahil) |
| `journalctl -f` | canlı takip |
| `journalctl -b [-N]` | belirli boot'un logları (`-1` = bir önceki boot) |
| `journalctl --file=<path>` | belirli bir journal dosyasını doğrudan oku |
| `journalctl --flush` | RAM'deki (`/run/log/journal`) logları diske (`/var/log/journal`) taşı |
| `journalctl --relinquish-var` | diski bırak, yeni yazımlar RAM'e gitsin |
| `journalctl --sync` | tüm açık journal dosyalarını elle diske indir (fsync) |
| `systemd-analyze cat-config systemd/journald.conf` | tüm config katmanlarının birleşmiş/etkin halini göster |
| `systemctl is-active rsyslog` | rsyslog çalışıyor mu |
| `strace -f -e trace=fsync,fdatasync -p $(pidof systemd-journald)` | journald'ın ne zaman fsync yaptığını canlı izle |

## 3.1 — Mimari: giriş kapıları → journald → dosya (+ syslog aynası)

### Diyagram 1 — temel akış: kaynaklar → journald → depolama

![journald mimarisi - seviye 1](misc/journald_architecture_mermaid.png)

En basit hal: giriş kapıları (kernel ring buffer, syslog(), native API, systemd stdout/stderr, kernel audit) journald'da toplanır, kalıcı (disk) veya geçici (RAM) depoya yazılır, ayarlıysa syslog/kmsg/console/wall'a iletilir. Mekanizma detayı yok, sadece akış.

### Diyagram 2 — + rate limiting, mmap, forward hedefinin ayrıntısı

![journald mimarisi - seviye 2](misc/journald_architecture_mermaid_2.png)

Bir önceki diyagrama eklenenler: journald'ın mesajlara filtre/rate limit uygulaması, journal dosyasına `mmap` ile erişim, ve forward hedefinin açılımı — rsyslog artık kendi gerçek dosyalarına yazan ayrı bir kutu (`/var/log/syslog`, `/var/log/auth.log`, `/var/log/kern.log`).

### Diyagram 3 — kapsamlı: config katmanları, durability, rotation, RAM ayrımı

![journald mimarisi - seviye 3](misc/journald_architecture_mermaid_3.png)

En kapsamlı hal. Ek olarak: config dosyalarının katmanları (`journald.conf` + drop-in'ler), `Storage=` değerlerinin dördü, shutdown'da `--smart-relinquish-var` ile RAM'e geri dönüş, durability zinciri (kernel writeback → `SyncIntervalSec` → CRIT+ mesajlarda anında sync), rotation/retention ayarları ve en kritik nokta: RAM'deki geçici journal (tmpfs, `/run/log/journal`) ile kalıcı dosyanın RAM'deki page cache'i FARKLI şeyler.

| `Storage=` değeri | Ne olur |
|---|---|
| `persistent` | disk yoksa oluşturulur, her zaman diske yazılır |
| `auto` (varsayılan) | `/var/log/journal` varsa diske, yoksa RAM'e |
| `volatile` | her zaman RAM (tmpfs), reboot'ta kaybolur |
| `none` | journal dosyasına hiç yazılmaz, sadece forwarding çalışır |

| Kritik ayrım | Anlamı |
|---|---|
| `--flush` vs `--sync` | flush: geçici journal → kalıcı journal geçişi; sync: yazılmış kayıtların diske inmesini (fsync) bekleme |
| `SyncIntervalSec` | "her kayıt bu süre kadar RAM'de bekler" demek değil — kernel writeback daha erken yazabilir |
| append-based ≠ append-only | kayıtlar ekleme şeklinde yazılır ama dosya başlığı ve indeksler güncellenebilir |

journald'a mesaj 4 farklı kapıdan girebilir: kernel ring buffer (`/dev/kmsg`), klasik `syslog()` çağrısı (`/dev/log`), native `sd_journal_send()` socket'i, ve unit'lerin stdout/stderr'ı. Hepsi journald'da toplanır, trusted field'lar eklenir (`_PID`, `_UID`, `_COMM`, `_SYSTEMD_UNIT`... — client bunları sahteleyemez), kendi binary `*.journal` dosyasına yazılır. `ForwardToSyslog=yes` ayarı açıksa aynı mesajın bir kopyası ayrıca rsyslog'a gider.

```
logger "mesaj"
   │ (syslog() çağrısı, /dev/log üzerinden)
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

Test: tek `logger` komutu, iki farklı sistemde iki farklı formatta (binary vs text) görünüyor mu?

```
$ logger "Naber journalctl!"
$ journalctl -n 3
Sep 24 20:38:20 lev-k ubuntu[1202]: Naber journalctl!
$ systemctl is-active rsyslog
active
$ sudo tail -3 /var/log/syslog
2026-09-24T20:40:08.081218+00:00 lev-k ubuntu: Naber journalctl!
```

Aynı mesaj iki tarafta da göründü. Kaynak drop-in:

```
$ cat /usr/lib/systemd/journald.conf.d/syslog.conf
[Journal]
ForwardToSyslog=yes
```

Sırayı doğrulamak için journald'ı atlayıp rsyslog'un dinlediği socket'e doğrudan yazmayı denedik:

```
$ echo "<13>socat test mesaji" | socat - UNIX-SENDTO:/run/systemd/journal/syslog
$ logger -u /run/systemd/journal/syslog "logger ile direkt socket testi"
$ tail -f /var/log/syslog
2026-09-24T20:56:12.871346+00:00 lev-k socat test mesaji
2026-09-24T20:57:41.352612+00:00 lev-k ubuntu: logger ile direkt socket testi
$ journalctl | grep -E "socat test|direkt socket testi"
                                            # (boş — hiçbir eşleşme yok)
```

| Yöntem | journalctl'de görünür mü | syslog'da görünür mü | Neden |
|---|---|---|---|
| `logger` (normal, `/dev/log`) | evet | evet (forward sayesinde) | önce journald'a girer |
| socket'e doğrudan yazma (`socat`/`logger -u`) | hayır | evet | journald'ı tamamen atlıyor, doğrudan rsyslog'un dinlediği socket'e düşüyor |

Bu, journald ile rsyslog'un iki bağımsız sistem olduğunun ilk kanıtı — biri diğerinin arkasında sessizce çalışmıyor, aralarında sadece tek yönlü bir kopyalama ilişkisi var.

## 3.2 — Dosya envanteri ve config katmanları

journald'a ait dosya/dizinler nerede duruyor (disk mi RAM mi), ayar değiştirmek istediğinde nereye yazman gerekiyor.

| Yol | Ne | Kalıcı mı |
|---|---|---|
| `/usr/lib/systemd/systemd-journald` | daemon binary | disk |
| `/usr/bin/journalctl` | client — dosyaları doğrudan okur, daemon'a sormaz | disk |
| `/etc/systemd/journald.conf` | admin ana config | disk |
| `/etc/systemd/journald.conf.d/*.conf` | admin drop-in | disk |
| `/run/systemd/journald.conf.d/*.conf` | runtime drop-in | RAM |
| `/usr/lib/systemd/journald.conf.d/*.conf` (`syslog.conf`) | vendor drop-in | disk |
| `/var/log/journal/<machine-id>/` | kalıcı log verisi | disk |
| `/run/log/journal/<machine-id>/` | volatile log verisi | RAM (tmpfs) |
| `/run/systemd/journal/{dev-log,socket,stdout,syslog}` | AF_UNIX socket'lar | RAM |

Config katman önceliği: ana dosya önce okunur, sonra tüm drop-in'ler alfabetik sırayla; **aynı isimli drop-in'de `/etc` > `/run` > `/usr/lib`** kazanır.

```
$ systemd-analyze cat-config systemd/journald.conf
# /etc/systemd/journald.conf
...
[Journal]
ForwardToSyslog=yes
```

`ForwardToSyslog` derleme zamanı default'unda `no` (ana dosyadaki `#ForwardToSyslog=no` satırı yorum, hiç etkili değil); etkin `yes` değeri vendor drop-in'inden (`/usr/lib/.../syslog.conf`) geliyor — Ubuntu'nun eski syslog pipeline'ıyla uyumluluk için bilinçli tercihi.

Kendi drop-in'imizi yazıp doğruladık:

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

İki drop-in yan yana, ikisi de etkin — anahtarlar çakışmadığı sürece katmanlar birleşir, sadece aynı anahtar birden fazla yerde tanımlıysa öncelik kuralı devreye girer.

Temizlik: `sudo rm /etc/systemd/journald.conf.d/99-lab.conf && sudo systemctl restart systemd-journald && sudo rmdir /etc/systemd/journald.conf.d/`

## 3.3 — Storage= ve flush akışı (RAM ↔ disk)

`Storage=` journald'ın nereye yazacağını belirler: `auto` (Ubuntu default'u) = `/var/log/journal` varsa oraya (kalıcı gibi davranır, dizini kendi oluşturmaz); `volatile` = sadece `/run/log/journal` (RAM), reboot'ta gider; `none` = hiç yazma, sadece forward et.

| Dizin | Storage | Mount kaynağı |
|---|---|---|
| `/var/log/journal/<machine-id>/` | Disk | `/dev/root` |
| `/run/log/journal/<machine-id>/` | RAM | `tmpfs` |

```
$ df -h /var /run
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        19G  2.8G   16G  15% /
tmpfs            83M  2.0M   82M   3% /run
```

Test: `Storage=volatile` drop-in'i yazıp restart, reboot öncesi/sonrası karşılaştır:

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

Reboot sonrası:

```
$ journalctl | grep "volatile reboot testi"
                                            # (boş)
$ grep "volatile reboot testi" /var/log/syslog
2026-09-25T14:06:37.908970+00:00 lev-k ubuntu: volatile reboot testi
```

`Storage=volatile` iken journal RAM'de (`/run/log/journal`) yaşıyor, reboot'ta sıfırlanıyor — mesaj journalctl'den gitti. `/var/log/syslog` etkilenmedi, çünkü rsyslog kendi bağımsız text dosyasına yazıyor ve journald'ın Storage ayarından habersiz.

`--flush` ve `--relinquish-var` bu iki modu **restart etmeden, canlıyken** birbirine çevirir:

| Flag | Yön | Ne yapar |
|---|---|---|
| `--flush` | RAM → Disk | `/run/log/journal`'daki birikmiş logları `/var/log/journal`'a taşır, sonraki yazımlar diske gider |
| `--relinquish-var` | Disk → RAM | diski bırakır, sonraki yazımlar RAM'e gider (diskteki eski kayıtlar silinmez) |

Canlı geçiş testi (önce temiz `persistent` durumuna dönüldü: `sudo rm /etc/systemd/journald.conf.d/99-lab.conf && sudo systemctl restart systemd-journald`):

**Test A — `--relinquish-var` öncesi/sonrası:**

```
$ logger "relinquish test A"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "relinquish test A"
Failed to open files: No such file or directory        # RAM'de yok — dizin henüz yok
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "relinquish test A"
Sep 25 21:31:32 lev-k ubuntu[13223]: relinquish test A  # disk'te var
$ grep "relinquish test A" /var/log/syslog
2026-09-25T21:31:32.261966+00:00 lev-k ubuntu: relinquish test A

$ sudo journalctl --relinquish-var
$ logger "relinquish test B"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "relinquish test B"
Sep 25 21:32:12 lev-k ubuntu[13237]: relinquish test B  # artık RAM'de
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "relinquish test B"
                                            # (boş — disk artık büyümüyor)
$ grep "relinquish test B" /var/log/syslog
2026-09-25T21:32:12.206210+00:00 lev-k ubuntu: relinquish test B
```

**Test B — `--flush` öncesi/sonrası:**

```
$ logger "flush test A"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "flush test A"
Sep 25 21:32:45 lev-k ubuntu[13245]: flush test A       # hâlâ RAM'de
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "flush test A"
                                            # (boş)

$ sudo journalctl --flush
$ logger "flush test B"
$ journalctl --file=/run/log/journal/$(cat /etc/machine-id)/system.journal | grep "flush test B"
Failed to open files: No such file or directory        # dizin artık yok
$ journalctl --file=/var/log/journal/$(cat /etc/machine-id)/user-1000.journal | grep "flush test B"
Sep 25 21:35:38 lev-k ubuntu[13278]: flush test B       # diske döndü
```

`/var/log/syslog` her iki testte de değişmedi — rsyslog Storage geçişlerinden tamamen bağımsız.

## 3.4 — Durability: fsync zamanlaması ve crash senaryosu

journald bir mesajı `mmap` ile önce page cache'e (RAM) yazar; gerçek diske iniş (`fsync`) hemen olmaz. `strace` ile journald process'ini izleyip hangi mesajların fsync'i beklettiğini, hangilerinin anında tetiklediğini gördük.

```
# terminal 1
$ sudo strace -f -e trace=fsync,fdatasync -p $(pidof systemd-journald)
strace: Process 13215 attached

# terminal 2
$ logger "strace normal mesaj"
```

Terminal 1'de hiçbir şey görünmedi.

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

| Gönderilen | Terminal 1'de görülen | Anlamı |
|---|---|---|
| `logger "normal mesaj"` | hiçbir şey | normal öncelik fsync'i bekletir, `SyncIntervalSec` timer'ına (default 5dk) bırakır |
| `logger -p crit "..."` | anında `fsync` | `crit`/`alert`/`emerg` mesajlar timer'ı beklemez, anında sync tetikler |
| `sudo journalctl --sync` | anında `fsync` (birden fazla dosya) | elle zorlama, tüm açık journal dosyalarını hemen diske indirir |

Crash testi: bir mesajı `--sync` ile garantiye alıp, bir mesajı garantisiz bırakıp makineyi `sysrq-trigger` ile anında (kapanış prosedürü olmadan) resetledik:

```
$ logger "crash test SYNCED"
$ sudo journalctl --sync
$ logger "crash test UNSYNCED" && echo b | sudo tee /proc/sysrq-trigger
                                            # bağlantı burada anında koptu
```

Reboot sonrası:

```
$ journalctl -b -1 | grep "crash test"
Sep 25 21:52:44 lev-k ubuntu[13329]: crash test SYNCED
$ grep "crash test" /var/log/syslog
                                            # (boş — HİÇBİRİ yok, SYNCED dahil)
```

| Mesaj | journald tarafında (`-b -1`) | syslog tarafında |
|---|---|---|
| `crash test SYNCED` (+ `--sync`) | kaldı | **kayboldu** |
| `crash test UNSYNCED` | kayboldu | kayboldu |

Beklenmedik ama önemli sonuç: `--sync` sadece journald'ın kendi binary dosyasını garantiye alıyor, rsyslog'un kendi text dosyasını sync'lemesini hiç zorlamıyor. rsyslog'un kendi bağımsız buffer/sync politikası var; crash anında henüz diske inmemişti, o da kayboldu. journald tarafında "garantili" olan bir mesaj bile syslog tarafında garantisiz kalabiliyor — iki sistemin gerçekten bağımsız olduğunun bir başka somut kanıtı.

## 3.5 — journalctl referansı (kalan komutlar)

Yukarıda gerçek çıktıyla test edilenlerin dışında, notlarda geçen ama case study yapılmamış ek filtreleme/format/temizlik flag'leri. Saf referans tablosu — bir kısmı (`--disk-usage`, `--vacuum-*`, `--rotate`) makinede denendi, kalanı denenmedi.

| Komut | Ne yapar |
|---|---|
| `journalctl --disk-usage` | Journal'ın toplam disk kullanımını gösterir |
| `sudo journalctl --vacuum-size=SIZE` | Arşiv dosyalarını boyut limitine göre siler — **aktif dosyaya asla dokunmaz**, o yüzden toplam kullanım limitin altına inmeyebilir. Silinen arşiv dosya adlarındaki `~` eki "kirli kapanmış" (unclean shutdown) işaretidir |
| `sudo journalctl --rotate` | Aktif dosyayı arşive çevirir, yeni aktif dosya açar — sessiz çalışır, etkisi `journalctl -u systemd-journald` ile doğrulanır |
| `journalctl -S <zaman>` / `-U <zaman>` | `--since`/`--until` kısa formu |
| `journalctl -b` / `-b -1` | belirli bir boot'un logları (`-1` = bir önceki boot) — `-b -1` crash testinde canlı kullanıldı (3.4) |
| `journalctl --list-boots` | kayıtlı tüm boot'ları listele |
| `journalctl _COMM=sshd` | field eşleşmesiyle filtrele (`_COMM` = trusted field, komut adı) |
| `journalctl _COMM=sshd + _COMM=autossh` | `+` ile iki field eşleşmesini OR'la |
| `journalctl --field=_COMM` | bir field'ın alabileceği tüm değerleri listele |
| `journalctl -o verbose` | her entry'nin tüm field'larını (trusted dahil) göster |
| `journalctl -o json` / `-o json -f` | JSON çıktısı, canlı takiple birlikte de kullanılabilir |
| `journalctl -o short-monotonic` | zaman damgasını boot'tan beri geçen süre olarak göster |
| `journalctl --header` | journal dosyasının kendi header bilgisini (format, boyut limitleri...) göster |

## Notlar

| Kavram | Not |
|---|---|
| `syslog` vs `rsyslog` | `syslog` bir protokol/API adı (RFC 5424, `syslog()` fonksiyonu), somut program değil; `rsyslog` Ubuntu'nun varsayılan somut implementasyonu (`r` = "rocket-fast", performans vurgusu) |
| journald ↔ rsyslog ilişkisi | tek yönlü kopyalama (`ForwardToSyslog=yes`), iki bağımsız sistem — biri diğerinin garantisini miras almaz (bkz. 3.4 crash testi) |
| `Storage=auto` (Ubuntu default) | `/var/log/journal` varsa oraya yazar (persistent gibi davranır), dizini kendi oluşturmaz |
| `SyncIntervalSec` | tek durability ayarı; `Seal` tamper tespiti içindir, `Compress` disk tasarrufu içindir — ikisinin de durability'ye etkisi yok |
| Rotation/retention (`SystemMaxUse`, `MaxRetentionSec`...) | bu bölüme girmedi, ayrı ele alınacak |

**Atlanan:** disk quota Bölüm 4'te değerlendirilmişti; rotation/retention ayrı bırakıldı (yukarıya bkz.).

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
