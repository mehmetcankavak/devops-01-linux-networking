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
