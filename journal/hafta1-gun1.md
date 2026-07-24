# Hafta 1, Gün 1 — inode ve Hard Link

## Öğrendiklerim
- Dosya adı = dizindeki bir kayıt, gerçek dosya = inode
- `ln a.txt b.txt` kopyalamaz, aynı inode'a ikinci isim ekler
- İki dosyanın aynı inode'u paylaştığını `ls -li` ile doğruladım (532130)
- `rm a.txt` sonrası `b.txt` veriyi korudu — link count 2→1 düştü, 0 olmadı
- mtime ≠ ctime: içerik değişmeden de ctime değişebilir (inode metadata değişikliği)

## Kanıt
\`\`\`
$ ls -li a.txt b.txt
532130 -rw-rw-r-- 2 ubuntu ubuntu 5 ... a.txt
532130 -rw-rw-r-- 2 ubuntu ubuntu 5 ... b.txt

$ rm a.txt && cat b.txt
veri
\`\`\`
## İzin Bitleri ve Kernel Kontrol Sırası
- Dizinlerde r/w/x farklı anlama gelir: x olmadan dosyaya isimle bile erişilemez (path traversal)
- `chmod 600 dizin` → `ls` çalışır, dosya içeriğine `cat` ile erişilemez (x yok)
- Kernel izin kontrolü SIRALI: EUID=dosya sahibiyse SADECE owner bitlerine bakılır, group/other hiç değerlendirilmez
- Kanıt: `chmod 077 dosya` (owner=000, group/other=777) → sahibi bile OKUYAMAZ

## umask
- gerçek_mode = istenen_mode & ~umask
- Kernel dosyalara asla otomatik execute biti vermez (dosya 666'dan, dizin 777'den başlar)
- umask 027 ile touch → 640 (666 & ~027), kanıtladım
- ÜRETİM TUZAĞI: systemd servisleri interaktif shell'in umask'ını miras almaz, UMask= ile ayrı tanımlanır.
  "Lokalde çalışıyor sunucuda 403" hatalarının klasik sebeplerinden biri.

## SUID / SGID / Sticky
- SUID(4): process dosya SAHİBİNİN yetkisiyle koşar (örn. /usr/bin/passwd, gerçek sistemde doğruladım). Script'lerde Linux SUID'i yok sayar.
- rws = SUID+execute birlikte; rwS (büyük S) = SUID var ama execute yok, şüpheli/hatalı konfigürasyon işareti
- Sticky(1): /tmp mode 1777, herkes yazar, sadece kendi dosyasını siler
- SGID(2) dizinde: içinde oluşan dosyalar OLUŞTURANIN değil DİZİNİN grubunu alır
- Kanıt: newgrp ile gid=deneme-grubu yaptım, yine de yeni dosya "ubuntu" (dizin grubu) aldı, SGID doğrulandı
- Güvenlik dersi: SUID script yerine C wrapper + göreli komut cagrisi, PATH hijack ile root shell riski. Dogru cozum: sudoers NOPASSWD tek komut ya da Linux capabilities (setcap)
