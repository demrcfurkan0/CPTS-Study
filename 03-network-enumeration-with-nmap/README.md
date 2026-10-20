[← Tüm modüller](../README.md)

# 03 — Network Enumeration with Nmap

HTB CPTS path'inin üçüncü modülü için tuttuğum çalışma notları. Nmap'in iç işleyişi: host discovery, port scanning state mantığı, TCP scan türleri (SYN/Connect/ACK/Window/Maimon/Null/FIN/Xmas/Idle), service & version detection, NSE, performans tuning, ve en kritik kısım **firewall + IDS/IPS evasion** (ACK scan, decoy, source port spoofing, DNS proxying, fragmentation).

> **Bu not neden farklı yazıldı:** Nmap zaten güçlü olduğum alan. O yüzden `nmap -sV -sC -p-` gibi temel komutları tekrar etmek yerine notu **state mantığı + scan türü seçimi + evasion refleksi** üzerine kurdum — yani "hangi senaryoda hangi flag ve neden". Günlük kullandığım komutlar burada değil; burada olanlar **sınavda/gerçekte takıldığım yerde dönüp bakacağım** şeyler.

Notlar Türkçe yazıldı, teknik terimler bilinçli olarak İngilizce bırakıldı — saha dili bu şekilde ve terimleri çevirmek aramayı zorlaştırıyor.

---

## İçindekiler

1. [Scan Türleri — Tam Liste](#1-scan-türleri--tam-liste)
2. [Host Discovery & ARP Tuzağı](#2-host-discovery--arp-tuzağı)
3. [Port State'leri — 6 Durum](#3-port-stateleri--6-durum)
4. [SYN vs Connect vs UDP — Paket Düzeyinde](#4-syn-vs-connect-vs-udp--paket-düzeyinde)
5. [Filtered: Dropped vs Rejected](#5-filtered-dropped-vs-rejected)
6. [Service & Version Detection](#6-service--version-detection)
7. [NSE — Scripting Engine](#7-nse--scripting-engine)
8. [Performance Tuning](#8-performance-tuning)
9. [Firewall & IDS/IPS Evasion](#9-firewall--idsips-evasion)
10. [Output & Raporlama](#10-output--raporlama)
11. [Debug Flag'leri — Ne Olduğunu Görmek](#11-debug-flagleri--ne-olduğunu-görmek)
12. [Tek Bakışta Karar Tablosu](#12-tek-bakışta-karar-tablosu)

---

## 1. Scan Türleri — Tam Liste

`nmap --help` çıktısındaki tüm teknikler — çoğu nadir kullanılır ama **evasion'da** işe yarar:

| Flag | Scan | Mantık | Ne zaman |
|---|---|---|---|
| **`-sS`** | TCP SYN (half-open) | SYN gönder, handshake'i tamamlama | **Default** (root). Hızlı + görece sessiz |
| **`-sT`** | TCP Connect | Tam 3-way handshake (`connect()`) | Root değilsek default; en doğru ama **en gürültülü** (log bırakır) |
| **`-sA`** | TCP ACK | Sadece ACK flag | **Firewall rule mapping** — filtered mi değil mi (open/closed ayırt etmez) |
| **`-sW`** | TCP Window | ACK scan + window field analizi | Bazı sistemlerde open/closed ayrımı |
| **`-sM`** | TCP Maimon | FIN/ACK | Bazı BSD türevi stack'lerde |
| **`-sN` / `-sF` / `-sX`** | Null / FIN / Xmas | Flag yok / FIN / FIN+PSH+URG | **Stateless firewall evasion** — RFC'ye göre closed port RST döner, open port sessiz kalır |
| **`-sU`** | UDP | Boş datagram | UDP servisler; çok yavaş (stateless, timeout uzun) |
| **`-sI`** | **Idle scan** | Zombie host'un IP ID'si üzerinden | **Tam anonim** — kendi IP'n hiç görünmez |
| **`-sY` / `-sZ`** | SCTP INIT / COOKIE-ECHO | SCTP protokolü | Telekom/SS7 ortamları |
| **`-sO`** | IP protocol | Hangi IP protokolleri destekli | — |
| **`-b`** | FTP bounce | FTP sunucusu üzerinden proxy | Eski/nadir |
| **`--scanflags`** | Custom | TCP flag kombinasyonunu elle belirle | IDS imza atlatma |

> **Sınav refleksi:** Firewall arkasında SYN scan filtered dönüyorsa → sırayla **ACK scan** (rule anla) → **source-port 53** → **Null/FIN/Xmas** (stateless firewall) → **idle scan** (tam gizlilik).

---

## 2. Host Discovery & ARP Tuzağı

```bash
nmap 10.129.2.0/24 -sn           # -sn: port scan kapalı, sadece "alive mi"
nmap -sn -iL hosts.lst           # IP listesinden
nmap -sn 10.129.2.18-20          # octet range
```

**En kritik gotcha — ARP ping:** `-sn` ICMP echo (`-PE`) yapar diye düşünürsün, ama **aynı local segmentte Nmap önce ARP ping atar**. ICMP hiç gönderilmeden ARP reply ile host "alive" işaretlenir.

```bash
nmap <ip> -sn -PE --packet-trace            # → "ARP who-has" görürsün, ICMP değil
nmap <ip> -sn -PE --disable-arp-ping --packet-trace   # ARP'ı kapat → gerçek ICMP echo
nmap <ip> -sn --reason                      # neden "up" dedi (arp-response / echo-reply)
```

| Flag | İş |
|---|---|
| `-PE` | ICMP echo request ile ping |
| `--disable-arp-ping` | ARP ping'i kapat (gerçek ICMP davranışını görmek için) |
| `--packet-trace` | Gönderilen/alınan her paketi göster |
| `--reason` | State'in sebebini göster |

> **Neden önemli:** Firewall ICMP'yi drop ediyorsa host "down" görünür — ama gerçekten offline mı, yoksa sadece ICMP mi filtreli? `-Pn` ile ping'i tamamen atlayıp doğrudan port tara.

---

## 3. Port State'leri — 6 Durum

| State | Anlamı | Nasıl tespit edilir |
|---|---|---|
| **open** | Bağlantı kuruldu (TCP/UDP/SCTP) | SYN-ACK döner |
| **closed** | Port kapalı | **RST flag** döner (host alive teyidi de verir) |
| **filtered** | Open mı closed mı belirsiz | Yanıt yok **veya** ICMP error code |
| **unfiltered** | Erişilebilir ama open/closed belirsiz | **Sadece ACK scan'de** (RST döner) |
| **open\|filtered** | Yanıt gelmedi | Firewall/packet filter olabilir (UDP'de sık) |
| **closed\|filtered** | Belirsiz | **Sadece idle (IP ID) scan'de** |

**Flag kısaltmaları** (`--packet-trace`'te): `S`=SYN · `SA`=SYN-ACK · `RA`=RST-ACK · `R`=RST · `A`=ACK · `P.`=PSH-ACK (banner geliyor).

---

## 4. SYN vs Connect vs UDP — Paket Düzeyinde

### SYN (-sS) — half-open
```
SENT  S         (biz → hedef: SYN)
RCVD  SA        → port OPEN   (SYN-ACK; sonra RST ile kapatırız, handshake tamamlanmaz)
RCVD  RA        → port CLOSED (RST-ACK)
(yanıt yok)     → port FILTERED
```
Root gerektirir (raw socket). Handshake tamamlanmadığı için log'a "bağlantı" düşmez → görece sessiz.

### Connect (-sT) — full handshake
Tam 3-way handshake (`connect()` syscall). **En doğru** state tespiti ama **en gürültülü** — çoğu sistemde connection log üretir, IDS kolay yakalar. "Polite" sayılır çünkü normal client gibi davranır, servise zarar vermez. Root yoksa default budur.

### UDP (-sU) — stateless
```
Boş datagram gönder:
  ICMP type 3 / code 3 (port unreachable)  → CLOSED
  Uygulama yanıtı gelirse                   → OPEN
  Başka ICMP error                          → open|filtered
  Hiç yanıt yok (retry sonrası)             → open|filtered
```
Handshake yok, ACK yok → **çok yavaş** (top 100 port ~98 sn). Admin'ler TCP'yi filtreleyip UDP'yi unutur → değerli.

---

## 5. Filtered: Dropped vs Rejected

Firewall paketi iki şekilde eler — ayırt etmek rule'u anlamayı sağlar:

| | Dropped | Rejected |
|---|---|---|
| Yanıt | **Yok** (sessiz) | **Var** (explicit) |
| TCP | — | RST flag döner |
| ICMP | — | Error code döner |
| Scan süresi | **Uzun** (retry × timeout, `--max-retries` default 10) | Kısa |

**ICMP error kodları** (rejected'te): Net/Host/Port/Proto Unreachable, Net/Host Prohibited.

> Süre ipucu: Bir port scan'i tek portta ~2 sn sürüyorsa (normal ~0.05 sn) → **dropped** (retry yapıyor). Hızlı ICMP unreachable dönüyorsa → **rejected**. İkisi de "host alive ama port korumalı" demek.

---

## 6. Service & Version Detection

```bash
nmap <ip> -p- -sV                    # tüm port + version
nmap <ip> -p- -sV --stats-every=5s   # her 5 sn ilerleme
nmap <ip> -p- -sV -v                 # port bulundukça anında göster
# tarama sırasında [Space Bar] → o anki durum
```

**Nasıl çalışır:** Önce **banner** okur (3-way handshake sonrası servis `PSH` flag'li paketle kendini tanıtır). Banner yetmezse **signature-based matching** → tarama uzar.

> **Kritik gotcha — Nmap her şeyi göstermez:** `-sV` otomatik çıktısı banner'daki bilgiyi kırpabilir. SMTP örneği: Nmap "Postfix smtpd" der ama banner'da `220 inlane ESMTP Postfix (Ubuntu)` yazıyor — **Ubuntu** bilgisini atlar. **Manuel teyit** et:
```bash
nc -nv <ip> 25                       # banner'ı kendi gözünle gör
sudo tcpdump -i eth0 host <me> and <target>   # trafiği yakala
```
OSCP zihniyeti: araca körü körüne güvenme, banner'ı elle doğrula.

---

## 7. NSE — Scripting Engine

Lua scriptleri, **14 kategori:**

| Kategori | Ne yapar | Risk |
|---|---|---|
| `auth` | Auth credential tespiti | — |
| `broadcast` | Broadcast ile host discovery | — |
| `brute` | Servise brute-force login | Gürültülü |
| `default` | `-sC` ile çalışanlar | Düşük |
| `discovery` | Erişilebilir servisleri değerlendir | — |
| `dos` | DoS zafiyeti testi | **Servise zarar** |
| `exploit` | Bilinen zafiyeti exploit et | **İntrusive** |
| `external` | Dış servis kullanır | Veri sızabilir |
| `fuzzer` | Fuzzing | Yavaş + riskli |
| `intrusive` | Hedefi olumsuz etkileyebilir | **Dikkat** |
| `malware` | Malware kontrolü | — |
| `safe` | Savunmacı, zararsız | Güvenli |
| `version` | `-sV` uzantısı | — |
| `vuln` | Zafiyet tespiti (vulners dahil) | — |

```bash
nmap <ip> -sC                                  # default scriptler
nmap <ip> --script <kategori>                  # kategori
nmap <ip> --script banner,smtp-commands        # tek tek
nmap <ip> -p 80 -sV --script vuln              # vuln tara (vulners → CVE listesi)
nmap <ip> -p 80 -A                             # -sV + -O + --traceroute + -sC hepsi
```

> **Uyarı:** `--script vuln`, `exploit`, `dos`, `intrusive` kategorileri **client izni olmadan çalıştırılmaz** — servise zarar verebilir / IDS tetikler. (01 modülündeki "do no harm" + exploitation prioritization ile aynı çizgi.)

---

## 8. Performance Tuning

| Flag | İş | Not |
|---|---|---|
| `-T 0..5` | Timing template | 0=paranoid ... 3=normal (default) ... 5=insane |
| `--min-rate <n>` | Saniyede min paket | Bandwidth biliniyorsa (whitebox/whitelisted) hızlandırır |
| `--max-retries <n>` | Retry sayısı (default 10) | `0` = yanıt yoksa atla → hızlı ama **host/port kaçırabilir** |
| `--initial-rtt-timeout` / `--max-rtt-timeout` | RTT timeout | Çok kısa → host'ları gözden kaçırırsın |
| `--min-parallelism <n>` | Eşzamanlı probe | — |

**Timing template'leri:**
```
-T 0 paranoid   -T 1 sneaky   -T 2 polite      ← yavaş, IDS evasion
-T 3 normal (default)
-T 4 aggressive -T 5 insane                     ← hızlı, IPS block riski
```

> **Trade-off kuralı:** Hızlandırmanın bedeli **doğruluk**. `--max-retries 0` veya düşük RTT, 2 host'u kaçırabilir. Blackbox'ta `-T 1/2` (sessizlik), whitelisted whitebox'ta `--min-rate` + `-T 4/5` (hız).

---

## 9. Firewall & IDS/IPS Evasion

> Modülün en değerli kısmı — OSCP/CPTS'te filtered port gördüğünde başvuracağın araç kutusu.

### Firewall vs IDS vs IPS
| | Ne yapar |
|---|---|
| **Firewall** | Rule'a göre paketi pass / drop / reject |
| **IDS** | Pasif izler, pattern/signature eşleşince **admin'e haber verir** |
| **IPS** | IDS + otomatik **önlem alır** (IP'ni bloklar) |

### IPS tespiti (sahada)
Birden çok **VPS/farklı IP** kullan. Tek host'tan agresif tara — o host bloklanırsa IPS var demektir, başka VPS'e geç. **Risk:** IP bloklanır → ISP'ye şikayet → tüm internet erişimin kapanabilir. O yüzden sessiz tara.

### ACK Scan (-sA) — Firewall Rule Mapping
Sadece ACK flag gönderir. Firewall'lar dışarıdan gelen SYN'i bloklar ama **ACK'i genelde geçirir** (içeriden mi başladı dışarıdan mı ayırt edemez).
```
SYN scan (-sS):  22 open, 21 filtered, 25 filtered
ACK scan (-sA):  22 UNFILTERED, 21 filtered, 25 filtered
                 ↑ 22'den RST döndü = firewall o portu geçiriyor (unfiltered)
                 ↑ 25'ten hiç yanıt yok = dropped
```
Open/closed ayırt etmez ama **hangi port firewall'dan geçiyor** onu söyler.

### Decoy (-D) — Kaynağı Gizle
IP header'a sahte IP'ler enjekte eder; gerçek IP'n rastgele araya karışır.
```bash
nmap <ip> -p 80 -sS -D RND:5          # 5 rastgele decoy + benim IP'm aralarında
nmap <ip> -p 80 -sS -D ip1,ME,ip2     # manuel decoy sırası
```
> **Dikkat:** Decoy'lar **alive olmalı** — yoksa SYN-flooding koruması servisi erişilemez yapar. ISP/router spoofed paketleri filtreleyebilir.

### Source IP / Interface Spoof (-S / -e)
Belirli subnet bloklanmışsa farklı kaynak IP dene:
```bash
nmap <ip> -p 445 -O -S 10.129.2.200 -e tun0   # kaynak IP spoof, tun0'dan gönder
```
Örnekte `-S` ile farklı IP kullanınca filtered port **open** döndü + OS detection çalıştı.

### Source Port Spoof (--source-port) — En Pratik Trick
Firewall belirli kaynak portlara güvenir (DNS=53, HTTP=80). Admin IDS/IPS'i düzgün filtrelemezse bu portlardan gelen paket **trusted** geçer.
```bash
nmap <ip> -p 50000 -sS --source-port 53       # filtered → OPEN oldu!
ncat -nv --source-port 53 <ip> 50000          # ncat ile de bağlan → 220 ProFTPd
```
> **Refleks:** Port filtered ama host alive → `--source-port 53` (veya 80/443) dene. Çoğu zaman firewall DNS trafiğine güvenir.

### DNS Proxying (--dns-server)
```bash
nmap <ip> --dns-server <ns1>,<ns2>            # DNS sorgularını belirtilen sunucudan yap
```
DMZ'deyken şirketin DNS sunucuları internetten gelenden daha **trusted** — iç ağ host'larıyla etkileşim için kullanılabilir. (DNS: UDP 53 normal sorgu, TCP 53 zone transfer / >512 byte.)

### Fragmentation (-f)
Paketleri küçük fragment'lara böler → imza tabanlı IDS'in paketi birleştirip eşleştirmesini zorlaştırır. (Modülde başlıkta geçer, temel evasion tekniği.)

---

## 10. Output & Raporlama

```bash
nmap <ip> -oN scan.nmap      # normal (insan okur)
nmap <ip> -oG scan.gnmap     # grepable (grep/cut ile parse)
nmap <ip> -oX scan.xml       # XML (araçlara feed / HTML'e dönüştür)
nmap <ip> -oA scan           # HEPSİ birden → scan.nmap/.gnmap/.xml
```

**XML → HTML rapor** (non-technical kitle için — 01'deki reporting ile bağlanır):
```bash
xsltproc scan.xml -o scan.html
```

**Grepable parse örnekleri:**
```bash
nmap -sn 10.129.2.0/24 | grep for | cut -d" " -f5      # alive IP listesi
cat scan.gnmap | grep "/tcp" | wc -l                   # açık port sayısı
```

> **Her taramayı kaydet** — farklı tool/scan farklı sonuç verir; karşılaştırma + dokümantasyon + rapor için şart (01 modülündeki activity log disiplini).

---

## 11. Debug Flag'leri — Ne Olduğunu Görmek

Bir sonuç kafanı karıştırdığında "kaputu açan" flag'ler:

| Flag | Ne gösterir |
|---|---|
| `--packet-trace` | Gönderilen/alınan **her paket** (SENT/RCVD, flag'ler) |
| `--reason` | State'in **sebebi** (syn-ack / arp-response / port-unreach) |
| `-v` / `-vv` | Verbosity — port bulundukça anında |
| `--stats-every=5s` | Periyodik ilerleme |
| `-n` | DNS resolution kapat (hız + temiz trace) |
| `-Pn` | Ping'i tamamen atla (ICMP filtreliyse) |

**"Temiz trace üçlüsü"** — davranışı izole etmek için: `-Pn -n --disable-arp-ping`. Bu üçü olmadan trace'te ARP/DNS/ICMP gürültüsü scan davranışını gizler.

---

## 12. Tek Bakışta Karar Tablosu

| Durum | Yap |
|---|---|
| Ağda kim alive? | `nmap -sn <range>` (ama ARP tuzağını unutma) |
| Host "down" ama emin değilim | `-Pn` ile ping atla, doğrudan port tara |
| Hızlı genel bakış | `-F` (top 100) veya `--top-ports=N` |
| Tam tarama | `-p- -sV` |
| Version Nmap'te eksik görünüyor | `nc -nv <ip> <port>` ile banner'ı elle gör |
| Port **filtered**, host alive | ① `-sA` (rule anla) → ② `--source-port 53` → ③ `-sN/-sF/-sX` → ④ `-sI` idle |
| IP'm bloklanıyor | `-D RND:5` decoy / `-S` source spoof / başka VPS |
| Sessiz olmam lazım | `-T 1/2`, decoy, source-port, fragmentation |
| Whitelisted + hız | `--min-rate 300 -T 4/5` |
| Sonuç kafa karıştırıcı | `--packet-trace --reason -Pn -n --disable-arp-ping` |
| CVE tara (izinliyse) | `-sV --script vuln` |
| Rapor | `-oA` → `xsltproc xml -o html` |

---

## Kaynaklar

Notlar aşağıdaki kaynaklardan derlendi:

- **HTB Academy** — Network Enumeration with Nmap (CPTS path, Module 03)
- **Nmap** — resmi dokümantasyon: host-discovery-strategies, port-scanning-techniques, performance-timing-templates, output, NSE docs (nsedoc)
- **tcpdump / ncat** — manuel banner grabbing & trafik analizi

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
