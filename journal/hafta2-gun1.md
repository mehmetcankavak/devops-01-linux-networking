# Hafta 2, Gün 1 — OSI/TCP-IP Temelleri & IP Subnetting

## 7 Adımlı Paket Yolculuğu (curl https://api.example.com)

### Öğrendiklerim
- Bir isteğin yolculuğu 7 adım: (1) DNS ile isim çözümleme (2) kernel routing kararı (3) ARP ile L2/MAC çözümleme (4) TCP 3-way handshake (5) TLS handshake (6) HTTP istek/yanıt (7) FIN/ACK kapanış, kapatan taraf TIME_WAIT'e girer
- Bu 7 adım aynı zamanda teşhis akış şeması: "site açılmıyor" dendiğinde hangi adımda kırıldığını bulmak = teşhisin tamamı, katman atlanmamalı

## IP & Subnetting

### Öğrendiklerim
- /prefix, 32 bitin kaçının ağ kaçının host olduğunu belirler; kullanılabilir host = 2^(32-prefix) - 2 (network + broadcast düşülür)
- Bir /22'yi 4 eşit parçaya bölmek = her parça bir /24 olur (1024/4=256 adres/parça)
- ip route show çıktısında longest prefix match kuralı geçerli: daha spesifik (uzun prefix'li) rota, daha genel olandan (default/0.0.0.0/0) her zaman kazanır

### Kanıt
- ipcalc 10.20.0.0/22 ile elle hesapladığımız 4x /24 bölünmesi (10.20.0.0/24, 10.20.1.0/24, 10.20.2.0/24, 10.20.3.0/24) doğrulandı: Network 10.20.0.0/22, HostMin 10.20.0.1, HostMax 10.20.3.254, Broadcast 10.20.3.255, 1022 kullanılabilir host

## Routing Table & 2-VM Gerçek Routing Testi

### Öğrendiklerim
- ip route show çıktısındaki her satır: default via <gateway> (bilinmeyen her şey buraya), X.X.X.0/24 scope link (yerel ağ, gateway'e gerek yok), host-özel rotalar (örn. gateway'in kendisi)
- net.ipv4.ip_forward=1 olmadan bir Linux makinesi kendi arayüzleri arasında paket yönlendirmeyi (router gibi davranmayı) reddeder
- ip route add ile eklenen özel bir rota, default route'tan daha spesifik olduğu için longest prefix match ile kazanır — aynı makine hem genel internet trafiğini normal gateway'den hem belirli bir alt ağı farklı bir "router" üzerinden yönlendirebilir
- ip route get, kernel'in gerçekte hangi rotayı seçeceğini doğrudan sorup öğrenmenin yolu — tabloyu elle yorumlamaya gerek yok

### Kanıt
- lab1 (devops-tutorial, 192.168.252.4) üzerine ikinci bir IP eklendi: sudo ip addr add 10.20.1.10/24 dev enp0s1 label enp0s1:1
- lab1'de net.ipv4.ip_forward=1 kalıcı olarak açıldı (/etc/sysctl.d/99-fwd.conf + sysctl --system)
- lab2 (devops-lab2, 192.168.252.5) üzerine rota eklendi: sudo ip route add 10.20.1.0/24 via 192.168.252.4 dev enp0s1
- lab2'den ping -c3 10.20.1.10: %0 paket kaybı — lab1 üzerinden başarıyla yönlendirildi
- ip route get 10.20.1.10 → via 192.168.252.4 (bizim özel rotamız) — kanıtlandı
- ip route get 8.8.8.8 → via 192.168.252.1 (normal gateway) — kontrast: genel trafik etkilenmedi, sadece 10.20.1.0/24 özel rotayı kullandı

## DNS Çözümleme & dig/curl Teşhisi (Gün 3)

### Öğrendiklerim
- Çözümleme sırası: /etc/nsswitch.conf (hosts: files dns) → /etc/hosts (statik override, DNS'ten önce gelir) → /etc/resolv.conf → systemd-resolved cache → recursive resolver (root → TLD → yetkili sunucu)
- /etc/resolv.conf modern Ubuntu'da yalan söyleyebilir: sadece 127.0.0.53 (yerel stub) gösterir, gerçek upstream DNS'i resolvectl status söyler
- ping ile DNS test edilmez: ping aynı anda DNS + ICMP + routing'i test eder, hangisinin kırıldığını ayırt edemez. Katman katman: DNS için dig, TCP için nc -zv, HTTP için curl -v
- dig +trace ile cache bypass edilip gerçek recursive zincir (root NS → TLD NS → domain'in kendi NS'i → A kaydı) görülebilir
- Aynı domain için farklı sorgularda farklı ama geçerli IP dönebilir (anycast/load-balancing, ör. Google)
- dig @8.8.8.8 ile kendi resolver'ını atlayıp doğrudan bir sunucuya sorarak "benim DNS ayarım mı domain'in kendisi mi bozuk" ayrımı yapılır
- curl --resolve, /etc/hosts'u kalıcı değiştirmeden DNS'i geçici bypass edip belirli bir backend'i test etmenin yolu (mavi/yeşil deploy doğrulaması)

### Kanıt
- cat /etc/resolv.conf → nameserver 127.0.0.53 (stub) / resolvectl status → Current DNS Server: 192.168.252.1 (gerçek sunucu, multipass gateway)
- dig +short google.com → 172.217.23.142, dig +trace google.com → root-servers.net → gtld-servers.net (.com) → ns1-4.google.com → A kaydı 192.178.24.14 (farklı ama geçerli IP, anycast kanıtı)
- dig google.com MX → smtp.google.com, dig google.com NS → ns1-4.google.com
- dig @8.8.8.8 google.com → doğrudan Google'ın public DNS'ine sorgu, farklı sunucudan aynı doğrulukta cevap
- /etc/hosts'a "192.168.252.5 myapp.test" eklendi → getent hosts myapp.test doğru çözdü → ping myapp.test gerçekten lab2'ye (192.168.252.5) gitti, %0 kayıp — /etc/hosts'un DNS'ten önce geldiği kanıtlandı
