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
