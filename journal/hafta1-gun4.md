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
