# Hafta 1, Gün 4 — Process Yaşam Döngüsü: fork/exec, Zombie, Orphan

## Öğrendiklerim
- fork() mevcut process'i kopyalar, ebeveyne çocuğun PID'ini, çocuğa 0 döner (CoW ile bellek gerçek kopyalanmaz)
- exec() çocuğun bellek imajını yeni programla değiştirir
- Zombie: çocuk bitti ama ebeveyn wait() çağırmadı, exit code kernel'de asılı kalır. Zombie KILL EDİLEMEZ (zaten ölü)
- Orphan: ebeveyn önce ölürse çocuk PID 1'e (systemd) evlatlık verilir, systemd wait() çağırdığı için zombie olmaz

## Kanıt
- python3 fork ile bilerek zombie ürettim: PID 2165, PPID 2158, STAT=Z
- kill -9 2165 denedim: HİÇBİR ETKİSİ olmadı, hâlâ Z
- Ebeveyn (2158) sleep(120) bitince kendiliğinden öldü → zombie (2165) PID 1'e evlatlık verildi
  ve systemd tarafından ANINDA reap edildi → ps çıktısında Z satırı tamamen kayboldu

## ps STAT Kodları & Load Average

### Öğrendiklerim
- STAT kodları: R (çalışıyor/çalışmaya hazır), S (uykuda, kesilebilir), D (uykuda, kesilemez — disk I/O bekliyor), Z (zombie), T (durduruldu)
- Load average CPU kullanım yüzdesi DEĞİL — R ve D durumundaki süreç sayısının üstel ağırlıklı hareketli ortalaması
- Üstel yumuşatma nedeniyle load average anlık sıçramaz, sabit yük altında 1dk ortalamasının gerçek değere yaklaşması ~1 dakika sürer
- 1dk > 15dk ise yük yükseliyor, 1dk < 15dk ise yük düşüyor demektir

### Kanıt
- stress-ng --cpu 2: 2 process STAT=R, %CPU ~99, ama sleep 2 sonrası load average sadece 0.17 (henüz yükselmemiş) → sleep 10 ile 0.35'e çıktığı gözlendi (üstel gecikme kanıtlandı)
- stress-ng --hdd 2 --hdd-bytes 256M: PID 9949 STAT=D, %wa=50.0, us+sy=~33.4, id=16.7 — CPU neredeyse boşta ama disk I/O bekleyen süreç yüzünden load average yine tırmanıyor (0.45)
- Sonuç: yüksek load + düşük CPU kullanımı = disk/I/O darboğazı şüphesi, teşhis sırası top → %wa → ps STAT=D

## Bellek Metrikleri & OOM Killer

### Öğrendiklerim
- free -h: `free` = hiç kullanılmamış bellek, `buff/cache` = disk cache (anında geri alınabilir), `available` = gerçek "ne kadar kullanılabilir" cevabı — izleme/alarm bu sütuna göre kurulmalı
- ps: VIRT = rezerve edilen sanal adres alanı (abartılı), RES/RSS = fiziksel RAM'de gerçekten tutulan kısım, SHR = paylaşılan kısım (RES toplamı gerçek toplamı vermez, PSS gerekir)
- OOM Killer sadece talep gerçekten fiziksel RAM+swap toplamını AŞTIĞINDA devreye girer — "available düşük" tek başına yetmez, kernel reclaim ile son ana kadar direnir
- oom_score_adj (-1000..+1000): kernelin "kimi öldüreceğim" kararında süreç bazlı ayarlanabilir öncelik. systemd unit'lerinde OOMScoreAdjust= ile tuning yapılır

### Kanıt
- stress-ng --vm 2 --vm-bytes 1500M (toplam 1.46GB talep, RAM 1.9GB): OOM tetiklenmedi, reclaim ile idare etti
- stress-ng --vm 3 --vm-bytes 2700M --vm-keep (toplam 2.7GB talep > 1.9GB RAM + 0 swap): dmesg'de gerçek OOM Killer olayları, "Out of memory: Killed process ... oom_score_adj:1000" — stress-ng kendi sürecini bilerek yüksek skora ayarlamış (kendini feda ediyor)
- Gerçek sistem süreçlerinde oom_score_adj: sshd=-1000 (asla öldürme, SSH erişimi kilitlenmesin diye), dockerd=-500 (yüksek koruma) — production'da systemd OOMScoreAdjust= ile önceden ayarlanmış
- --vm-bytes değeri worker başına DEĞİL toplam hedef, worker sayısına bölünüyor (stress-ng log satırından doğrulandı: "using 300MB per stressor instance (total 900MB...)")
