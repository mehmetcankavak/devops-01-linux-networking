# Runbook: VM Saat Kayması (Clock Drift)

## Belirti
GitHub contribution grafiğinde güncel gün (28 Temmuz) hiç işaretlenmemiş görünüyordu,
"No contributions on July 28th" uyarısı çıkıyordu — oysa o gün aktif olarak commit atılmıştı.

## Teşhis
VM içinde `date` çalıştırıldığında gerçek tarihten 4 gün geride, 24 Temmuz'da donmuş
olduğu görüldü. VM'de atılan TÜM commit'ler bu yüzden gerçek günden bağımsız olarak
eski tarihi taşıyordu.

## Kök Neden
Multipass VM'i, host makine (laptop) uyku moduna geçtiğinde/uzun süre arka planda
kaldığında saatini host ile senkronize edemedi; NTP senkronizasyonu kopmuş olmalı.

## Çözüm
multipass restart devops-tutorial
VM yeniden başlatılınca Multipass, VM'in saatini host ile otomatik senkronize etti.

## Doğrulama
date
timedatectl status
→ `System clock synchronized: yes`, `NTP service: active`, tarih doğru.

## Önlem
- Uzun süre kullanılmamış/host'u uyutulmuş bir VM'de çalışmaya başlamadan önce `date`
  ile saat kontrolü alışkanlık hâline getirilmeli
- `timedatectl status` çıktısında "System clock synchronized: no" görülürse hemen müdahale
