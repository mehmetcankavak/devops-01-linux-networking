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

## systemd & cgroups — Kaynak Limiti Uygulaması

### Öğrendiklerim
- systemd her .service unit'ini bir cgroup'a bağlar (system.slice altında); systemctl status çıktısındaki Memory/CPU/Tasks değerleri systemd'nin uydurması değil, doğrudan /sys/fs/cgroup/.../memory.current ve cpu.stat dosyalarından okunuyor (birebir doğrulandı: memory.current=11206656B ≈ 10.6M, cpu.stat usage_usec=67418 ≈ 67ms)
- Unit dosyasında MemoryMax= koyarak cgroup'a gerçek, kernel seviyesinde zorlayıcı bir bellek tavanı konabilir — bu sadece izleme değil, gerçek bir sınır
- Aşım anında devreye giren cgroup-scoped OOM Killer sadece o servisin süreçlerini hedefler, sistem genelini etkilemez; systemd bunu "Failed with result 'oom-kill'" olarak işaretler
- Python stdout bir pipe'a (terminal değil) yazarken tam tamponlu (fully buffered) çalışır — SIGKILL ile aniden öldürülen sürecin tamponundaki loglar hiç flush edilmeden kaybolur (production'da "eksik log" tuzaklarından biri)

### Kanıt
- memory-eater.sh: sonsuz döngüde saniyede ~10MB bellek ayıran python script
- /etc/systemd/system/memory-eater.service: MemoryMax=100M, MemoryAccounting=yes
- journalctl -u memory-eater -f çıktısı: "The kernel OOM killer killed some processes in this unit" / "Failed with result 'oom-kill'" / "Consumed... 100M memory peak" — limit tam olarak MemoryMax değerinde tetiklendi
- Script'in print() çıktıları journal'da hiç görünmedi — buffering + ani SIGKILL nedeniyle kayboldu

## Sinyaller (Signals) & Exit Code Kuralı

### Öğrendiklerim
- SIGTERM ve SIGINT (Ctrl+C) yakalanabilir (trap) veya görmezden gelinebilir — süreç kendi temiz kapanış kodunu çalıştırma fırsatı bulur
- SIGKILL (-9) ve SIGSTOP/SIGCONT KESİNLİKLE yakalanamaz — bunlar kernel seviyesinde doğrudan uygulanır, sürecin trap kodu hiç çalışma fırsatı bulamaz (SIGKILL için süreç anında yok olur, SIGSTOP/CONT'ta süreç donar/devam eder ama kendi haberi bile olmaz)
- SIGSTOP ile durdurulan süreç STAT=T olur, hiç CPU harcamaz ama bellekte/PID tablosunda kalır — SIGCONT ile kaldığı yerden devam eder
- Exit code kuralı: bir sinyal tarafından öldürülen sürecin exit kodu 128+sinyal_no olur (SIGKILL=9 → 137, SIGTERM=15 → 143 vb.) — bu kural, $? veya systemctl çıktısından "hangi sinyal öldürdü" diye geriye doğru teşhis etmeyi sağlar
- Normal (trap ile) çıkışta exit code, script'in exit ile verdiği değerdir (bizim örnekte exit 0)

### Kanıt
- signal-test.sh: SIGTERM'i yakalayıp temiz çıkan (exit 0), SIGINT'i yakalayıp görmezden gelen bir script
- kill <PID> (SIGTERM): trap çalıştı, "temiz kapanıyorum" mesajı basıldı, $?=0
- kill -9 <PID> (SIGKILL): trap ÇALIŞMADI, sadece bash'in kendi "Killed" mesajı çıktı, $?=137 (128+9 doğrulandı)
- kill -STOP <PID> → ps STAT=T (donduruldu) → kill -CONT <PID> → ps STAT=S (sessizce devam etti, script'in haberi olmadı)

## nice/renice/ionice — Kaynak Önceliklendirme

### Öğrendiklerim
- nice değeri -20 (en yüksek öncelik) ile +19 (en düşük öncelik) arasında, CPU scheduler'ının sürece ne sıklıkla zaman vereceğini etkiler; normal kullanıcı sadece önceliğini düşürebilir (pozitif), yükseltmek (negatif) root ister
- nice komutu süreci başlatırken öncelik atar, renice zaten çalışan bir sürecin önceliğini sonradan değiştirir
- Nice değeri sadece GERÇEK CPU KITLIĞI altında etkili olur — CPU sayısından az süreç varken hiçbir fark yaratmaz (her süreç kendi çekirdeğini alır)
- ionice, CPU'dan tamamen ayrı bir mekanizma; disk I/O sırasını önceliklendirir. 3 sınıf: real-time (en yüksek, tehlikeli), best-effort (varsayılan, 0-7 seviye), idle (sadece disk boşken çalışır, diğer I/O'ya her zaman yol verir)

### Kanıt
- 2 CPU'lu VM'de 2 stress-ng süreci (biri nice=19, biri nice=-5): ikisi de %CPU=100 aldı, TIME+ birebir eşit — kıtlık yokken nice etkisiz
- Aynı VM'de 8 stress-ng süreci (4x nice=19, 4x nice=-5) 2 çekirdeğe sıkıştırılınca: nice=-5 süreçler ~%50 CPU (2 tam çekirdek), nice=19 süreçler ~%0.3 CPU (açlık) — gerçek kıtlıkta nice çok belirgin
- ionice -c 3 (idle) ile normal (best-effort, prio 0) iki stress-ng --hdd süreci yarıştırıldı: idle süreç %CPU=18.8/TIME+2.31s, best-effort süreç %CPU=22.0/TIME+2.53s — idle süreç beklendiği gibi daha az iş yapabildi (disk I/O'da geri planda kaldı)

## LAB 4 — Self-Healing systemd Service

### Öğrendiklerim
- Restart=on-failure: servis sıfırdan farklı exit code ile bittiğinde systemd otomatik yeniden başlatır
- RestartSec=N: yeniden başlatmadan önce N saniye bekletir, anlık deli-döngüyü önler
- StartLimitIntervalSec + StartLimitBurst: belirtilen zaman penceresinde izin verilen maksimum başlatma denemesi sayısını sınırlar; aşılırsa systemd pes eder, servisi kalıcı "failed" bırakır — production'daki crash-loop koruma mekanizmasının aynısı
- Her restart tamamen yeni bir süreç (yeni PID) yaratır, eskisini diriltmez

### Kanıt
- flaky-service.sh: 5 saniye çalışıp exit 1 ile bilerek çöken script
- flaky.service: Restart=on-failure, RestartSec=2, StartLimitIntervalSec=30, StartLimitBurst=4
- journalctl -u flaky -f: PID her seferinde farklı (9228→9363→15445→25599), "restart counter is at 1,2,3" sayaç arttı, 4. denemede "Start request repeated too quickly" ile durdu
- systemctl status flaky: Active: failed (Result: exit-code) — kalıcı durdu, insan müdahalesi bekliyor

## LAB 5 — Resource-Hog Avlama (Section 2 Kapanış Lab'ı)

### Öğrendiklerim
- %CPU/%MEM tek başına "kim tüketiyor" sorusuna cevap verir ama "nereden geldi" sorusuna vermez — gerçek olay müdahalesinde PPID zinciri (ps -fp / ps -o pid,ppid,cmd) yukarı doğru takip edilerek kökene inilmeli
- TTY sütunu (pts/0 vs ?) bir sürecin interaktif bir terminalden mi yoksa arka plan servisinden/cron'dan mı geldiğini ayırt etmeye yarar — şüpheli/kötücül süreç ayrımında kritik ipucu
- Bir süreç ailesini (parent+children) doğru temizlemek için sadece üstteki PID'i öldürmek yetmez, çocuklar öksüz kalıp çalışmaya devam edebilir — pkill -P <ppid> tüm çocukları hedefler
- Müdahale kararı bulgulara göre verilir: zaten düşük öncelikli (nice) ve kendi kendine sonlanan (timeout'lu), meşru kökenli bir süreç her zaman hemen öldürülmesi gereken bir tehdit değildir — bağlam önemli

### Kanıt
- nohup bash -c 'nice -n 10 stress-ng --cpu 1 --vm 1 --vm-bytes 300M --timeout 300s' & disown ile gizli süreç başlatıldı
- top: PID 1884 (stress-ng-vm, RES≈302MB, %MEM=15.5) ve PID 1882 (stress-ng-cpu, %CPU=100) şüpheli olarak tespit edildi
- ps -fp zinciri: 1882/1884 → PPID 1872/1883 → PPID 1836 (-bash, TTY=pts/0) — kökenin interaktif terminal olduğu doğrulandı, cron/gizli servis değil
- pkill -P 1872 ile tüm süreç ailesi tek seferde temizlendi, ps aux | grep stress-ng ile artık kalmadığı doğrulandı

## Section 2 — Process & Kaynak Yönetimi: TAMAMLANDI
