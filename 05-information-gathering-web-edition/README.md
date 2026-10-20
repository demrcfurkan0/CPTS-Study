[← Tüm modüller](../README.md)

# 05 — Information Gathering · Web Edition

HTB CPTS path'inin beşinci modülü için tuttuğum çalışma notları. Konu: **web reconnaissance** — bir web uygulaması hakkında active + passive bilgi toplama. WHOIS, DNS, subdomain enum/brute, zone transfer, virtual host fuzzing, certificate transparency, fingerprinting (banner/header/WAF), crawling, robots.txt, well-known URIs, Google Dorking, Wayback Machine ve automation framework'leri.

> **Not stratejisi:** DNS/cert-transparency kısımları 04-Footprinting ile örtüşüyor; onları özetledim. Asıl ağırlığı **web'e özgü** tekniklere verdim: VHost fuzzing (subdomain'den farkı kritik), fingerprinting zinciri (curl -I → redirect chain → WAF), Google Dork operatörleri, well-known URIs, Wayback. Her teknik için "ne komut + ne buluyorum + risk (detection)" bir arada.

Notlar Türkçe, teknik terimler İngilizce bırakıldı.

---

## İçindekiler

1. [Active vs Passive Recon](#1-active-vs-passive-recon)
2. [WHOIS](#2-whois)
3. [DNS — Kayıt Tipleri & dig](#3-dns--kayıt-tipleri--dig)
4. [Subdomain Enumeration & Brute-Force](#4-subdomain-enumeration--brute-force)
5. [DNS Zone Transfer](#5-dns-zone-transfer)
6. [Virtual Host (VHost) Fuzzing](#6-virtual-host-vhost-fuzzing)
7. [Certificate Transparency](#7-certificate-transparency)
8. [Fingerprinting — Teknoloji Tespiti](#8-fingerprinting--teknoloji-tespiti)
9. [Crawling & robots.txt & Well-Known URIs](#9-crawling--robotstxt--well-known-uris)
10. [Search Engine Discovery / Google Dorking](#10-search-engine-discovery--google-dorking)
11. [Web Archives — Wayback Machine](#11-web-archives--wayback-machine)
12. [Automating Recon](#12-automating-recon)
13. [Tek Bakışta: Web Recon Akışı](#13-tek-bakışta-web-recon-akışı)

---

## 1. Active vs Passive Recon

Web recon = sistematik bilgi toplama. 4 hedef: **asset tespiti** (sayfa/subdomain/IP/teknoloji), **gizli bilgi** (backup/config/doc), **attack surface analizi**, **intelligence** (kişi/email/pattern).

| | Active | Passive |
|---|---|---|
| Etkileşim | Hedefle **doğrudan** | 3rd-party / public kaynak |
| Detection | Orta-Yüksek | Çok düşük |
| Kapsam | Daha derin | Sadece public olanla sınırlı |

**Active teknikler (detection riski ↑):** port scan (Nmap, high), vuln scan (Nessus/Nikto, high), network mapping (traceroute), banner grabbing (nc/curl, low), OS fingerprinting, service enum, web spidering.
**Passive teknikler (detection ~yok):** search engine queries (Shodan), WHOIS, DNS (dig/host), Wayback, social media (LinkedIn), code repos (GitHub).

> OSINT, enumeration'dan ayrı — sadece passive. Önce passive ile zemin kur, active'i hedefli kullan (gürültü = blacklist riski).

---

## 2. WHOIS

Domain registration bilgisi. `sudo apt install whois`
```bash
whois facebook.com
```
**Okunan istihbarat:**
- **Creation/Expiry date** → yeni kayıt = şüpheli (phishing), uzak expiry = köklü/meşru
- **Registrar** → lax abuse policy'li registrar = kötücül ipucu
- **Registrant org/contact** → sahip kim (privacy service arkasında gizliyse red flag)
- **Name servers** → hosting provider; bulletproof hosting = malicious
- **Domain status** → `clientTransferProhibited` vb. (koruma seviyesi)

> **3 kullanım senaryosu:** phishing investigation (yeni kayıt + gizli registrant + bulletproof NS = kampanya), malware C2 analizi (anonim email + cybercrime-yoğun ülke), threat intel (domain cluster'ları, ortak NS → TTP profili + IOC).

---

## 3. DNS — Kayıt Tipleri & dig

DNS = internet GPS (domain → IP). Hierarchy: **Root → TLD (.com) → Authoritative NS**. Resolver recursive sorar, cache'ler.

**hosts dosyası** (DNS bypass, manuel map):
```
# Linux/Mac: /etc/hosts   Windows: C:\Windows\System32\drivers\etc\hosts
<IP> <hostname> [alias]
# VHost DNS kaydı yoksa buraya ekleyip erişilir — VHost testi için kritik
```

**Kayıt tipleri:**
| Tip | Ne | Recon değeri |
|---|---|---|
| A / AAAA | IPv4 / IPv6 | asset tespiti |
| CNAME | alias | eski/zafiyetli server'a point edebilir (subdomain takeover) |
| MX | mail server | email altyapısı |
| NS | authoritative NS | hosting provider |
| TXT | serbest metin | **SPF/DMARC/DKIM + 3rd-party verification** (`_1password=...` → SE ipucu) |
| SOA | zone admin bilgisi | primary NS + admin email |
| SRV | servis host+port | — |
| PTR | reverse (IP→host) | — |

**dig komutları:**
```bash
dig domain.com                  # default A
dig domain.com MX / NS / TXT / SOA / CNAME / AAAA
dig @1.1.1.1 domain.com         # belirli NS'e sor
dig +trace domain.com           # tam çözümleme yolu (root→TLD→auth)
dig -x 192.168.1.1              # reverse lookup
dig +short domain.com           # sadece cevap
dig +noall +answer domain.com   # sadece answer section
dig domain.com ANY              # tüm kayıtlar (RFC 8482 ile çoğu server ignore eder)
```
Diğer araçlar: `nslookup` (basit), `host` (kısa çıktı), dnsenum/fierce/dnsrecon (otomasyon), theHarvester (OSINT email).

---

## 4. Subdomain Enumeration & Brute-Force

İki yol: **passive** (cert transparency, search engine) + **active brute-force** (wordlist).

**Brute-force akışı:** wordlist seç (general/targeted/custom) → her kelimeyi domain'e ekle → DNS lookup (A/AAAA) → resolve edenleri topla.

```bash
# dnsenum — brute + zone transfer denemesi + Google scraping + reverse + WHOIS
dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r
#  -r = recursive (bulunan subdomain'in subdomain'lerini de dene)
```
Araçlar: **dnsenum, fierce, dnsrecon, amass** (en kapsamlı, çok kaynak), **assetfinder** (hızlı), **puredns** (güçlü brute).

> Wordlist seçimi: bilinmiyorsa general (`subdomains-top1million-*`), biliniyorsa targeted/custom. SecLists `Discovery/DNS/`.

---

## 5. DNS Zone Transfer

Primary→secondary NS senkronizasyonu (AXFR, TCP 53). `allow-transfer` yanlış yapılandırılmışsa → **tüm zone dosyası dökülür** (iç hostlar, IP'ler).
```bash
dig axfr inlanefreight.com @<nameserver>
dig axfr internal.inlanefreight.com @<ns>    # iç zone varsa → DC, VPN, WSUS...
```
> 04-Footprinting'deki ile aynı teknik — misconfig'li server'da jackpot. Modern server'larda genelde kapalı ama her zaman dene.

---

## 6. Virtual Host (VHost) Fuzzing

> **Bu modülün en kritik web-spesifik tekniği.** Subdomain ≠ VHost farkını bil.

**Nasıl çalışır:** Web server (Apache/Nginx/IIS) tek IP'de birden çok site barındırır. Hangi içeriği döneceğini **HTTP `Host` header**'ından anlar (name-based virtual hosting).

| | Subdomain | Virtual Host |
|---|---|---|
| DNS kaydı | **VAR** (public) | olmayabilir (iç/gizli) |
| Tespit | DNS enum, cert transparency | **Host header fuzzing** |
| Erişim | normal | DNS yoksa `/etc/hosts`'a ekle |

**Kritik nokta:** DNS'te görünmeyen (public olmayan) subdomain/VHost'lar sadece Host header değiştirerek bulunur. Bir IP'ye farklı Host header'ları göndererek response'u analiz et.

**VHost hosting tipleri:** name-based (Host header, en yaygın), IP-based (her site ayrı IP), port-based (farklı port).

```bash
# Gobuster vhost mode
gobuster vhost -u http://<target_IP> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain
#  --append-domain = kelimeye base domain ekle (yeni Gobuster'da ZORUNLU)
#  -t threads, -k SSL hata ignore, -o output dosyası
gobuster vhost -u http://inlanefreight.htb:81 -w <wordlist> --append-domain
# → Found: forum.inlanefreight.htb:81 Status: 200
```
Araçlar: **gobuster**, **feroxbuster** (Rust, hızlı, recursion), **ffuf** (Host header fuzzing).

> DNS enum ile VHost fuzzing **farklı sonuçlar** verir — ikisini de çalıştır. IDS/WAF tetikleyebilir, dikkat.

---

## 7. Certificate Transparency

SSL/TLS cert'lerinin audit-proof log'u (RFC 6962). Issue edilen her cert loglanır → subdomain madeni.
```bash
# crt.sh — JSON çıktı + unique subdomain filtre
curl -s "https://crt.sh/?q=inlanefreight.com&output=json" | jq . \
  | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 \
  | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```
Web arayüzü: `https://crt.sh/?q=<domain>`. Passive, detection yok. (04-Footprinting ile aynı araç.)

---

## 8. Fingerprinting — Teknoloji Tespiti

Hedefin teknoloji stack'ini çıkar → hedefli exploit + misconfig/outdated tespiti + öncelik.

**4 teknik:** banner grabbing · HTTP header analizi (`Server`, `X-Powered-By`) · probing (özel request → karakteristik response) · page content (copyright, path pattern).

**Banner grabbing + redirect chain takibi (manuel, çok öğretici):**
```bash
curl -I inlanefreight.com
# → Server: Apache/2.4.41 (Ubuntu) · 301 → https://inlanefreight.com/
curl -I https://inlanefreight.com
# → X-Redirect-By: WordPress · 301 → https://www.inlanefreight.com/
curl -I https://www.inlanefreight.com
# → Link: .../wp-json/  (wp- prefix = WordPress doğrulandı)
```
> **Ders:** Redirect zincirini sonuna kadar takip et — her hop yeni bilgi (WordPress, path'ler) sızdırır.

**Araçlar:**
| Araç | İş |
|---|---|
| **Wappalyzer** / **BuiltWith** | teknoloji profili (CMS, framework, analytics) |
| **WhatWeb** | CLI fingerprint (geniş imza DB) |
| **Netcraft** | hosting + security raporu |
| **wafw00f** | WAF tespiti (tip + config) |
| **Nikto** | web server vuln + fingerprint |

```bash
wafw00f inlanefreight.com         # → "behind Wordfence (Defiant) WAF"
nikto -h inlanefreight.com -Tuning b    # -Tuning b = sadece Software Identification
```
> **WAF tespiti önce yapılır** — probe'ları engelleyebilir/bloklayabilir, teknik adapte edilmeli. Nikto çıktısı: outdated software (Apache 2.4.41 eski), WordPress login (`/wp-login.php`), `license.txt`, eksik security header'lar (HSTS, X-Content-Type-Options).

---

## 9. Crawling & robots.txt & Well-Known URIs

### Crawling (spidering)
Seed URL → fetch → link çıkar → queue → tekrarla. İki strateji: **breadth-first** (geniş, yapı genel bakış) vs **depth-first** (derin, spesifik içerik).

> **Fuzzing'den farkı:** crawling var olan linkleri takip eder; fuzzing muhtemel linkleri tahmin eder.

**Çıkarılan değerli veri:** internal/external linkler (yapı haritası), **comments** (sensitive detay sızar), metadata (title/author/date), **sensitive files** (`.bak`/`.old`, `web.config`/`settings.php`, `error_log`). Araçlar: Burp/ZAP Spider, Scrapy.

> **Context kritik:** tek veri anlamsız olabilir; linklerde `/files/` pattern'i + "file server" comment'i birleşince → directory browsing açık, backup/doc exposure. Noktaları birleştir.

### robots.txt
Root'ta (`/robots.txt`), Robots Exclusion Standard. Crawler'lara "nereye girme" der.
```
User-agent: *
Disallow: /admin/        # ← recon için: gizli dizin ipucu!
Disallow: /private/
Allow: /public/
Crawl-delay: 10
Sitemap: https://example.com/sitemap.xml
```
> **Recon değeri:** `Disallow` path'leri = sahibinin gizlemek istediği yerler (admin panel, backup, config). Enforce edilmez — sadece ipucu. Honeypot/crawler trap da olabilir.

### Well-Known URIs
`/.well-known/` standart dizin (RFC 8615), IANA registry. Metadata/config merkezi.
| URI | İçerik |
|---|---|
| `security.txt` | güvenlik iletişim (RFC 9116) |
| `change-password` | parola değiştirme URL'i |
| **`openid-configuration`** | OpenID Connect config (JSON) |
| `assetlinks.json` | digital asset ownership |
| `mta-sts.txt` | SMTP MTA-STS policy |

```bash
curl https://example.com/.well-known/openid-configuration
# → authorization_endpoint, token_endpoint, userinfo_endpoint, jwks_uri,
#   scopes_supported, signing algs → OAuth/OIDC attack surface haritası
```

---

## 10. Search Engine Discovery / Google Dorking

Search engine = OSINT madeni. Public, legal, ücretsiz, geniş kapsam.

**Search operatörleri:**
| Operatör | İş | Örnek |
|---|---|---|
| `site:` | domain kısıtla | `site:example.com` |
| `inurl:` | URL'de terim | `inurl:login` |
| `filetype:` / `ext:` | dosya tipi | `filetype:pdf` |
| `intitle:` | başlıkta | `intitle:"confidential report"` |
| `intext:` | body'de | `intext:"password reset"` |
| `cache:` | cache'li versiyon | `cache:example.com` |
| `link:` / `related:` | link veren / benzer site | |
| `allintext:`/`allinurl:`/`allintitle:` | hepsi birden | `allinurl:admin panel` |
| `AND` / `OR` / `NOT` / `-` | mantık | `site:bank.com -inurl:login` |
| `*` | wildcard | `user* manual` |
| `..` | sayı aralığı | `"price" 100..500` |
| `" "` | tam ifade | `"information security policy"` |

**Klasik Google Dork'lar:**
```
site:example.com inurl:login                    # login sayfaları
site:example.com filetype:pdf                   # exposed dosyalar
site:example.com inurl:config.php               # config dosyaları
site:example.com (ext:conf OR ext:cnf)
site:example.com inurl:backup                   # DB backup
site:example.com filetype:sql
```
> Daha fazlası: **Google Hacking Database (GHDB)**.

---

## 11. Web Archives — Wayback Machine

Internet Archive'ın web snapshot arşivi (1996'dan beri). Site geçmişini görmeyi sağlar. 3 adım: crawl → archive (tarih+saat'le) → access.

**Recon değeri:**
- **Gizli asset/zafiyet:** mevcut sitede olmayan eski sayfa/dizin/dosya/subdomain → sensitive bilgi
- **Değişim takibi:** snapshot'ları karşılaştır → yapı/teknoloji/zafiyet evrimi
- **OSINT:** geçmiş aktivite, pazarlama, çalışan, teknoloji seçimleri
- **Stealth:** passive — hedefin altyapısıyla etkileşim YOK, tespit edilmez

> URL + tarih seç → eski haline bak. Site exclude talep etmemişse geçmiş erişilebilir.

---

## 12. Automating Recon

Manuel yavaş + hataya açık. Automation: efficiency, scalability, consistency, comprehensive coverage, integration.

**Framework'ler:**
| Araç | Odak |
|---|---|
| **FinalRecon** | Python, modüler (header/whois/ssl/crawl/dns/sub/dir/wayback/portscan) |
| **Recon-ng** | güçlü modüler framework, exploit entegrasyonu |
| **theHarvester** | email/subdomain/host/çalışan (search engine + Shodan + PGP) |
| **SpiderFoot** | OSINT otomasyon, çok veri kaynağı |
| **OSINT Framework** | araç/kaynak koleksiyonu |

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git && cd FinalRecon
pip3 install -r requirements.txt && chmod +x ./finalrecon.py

./finalrecon.py --headers --whois --url http://inlanefreight.com
./finalrecon.py --full --url http://inlanefreight.com    # hepsi
#  --headers --sslinfo --whois --crawl --dns --sub --dir --wayback --ps
#  -e txt,xml,php  (extension)  -k shodan@key  (API)  -o export formatı
```
> FinalRecon subdomain için crt.sh/AnubisDB/ThreatMiner/CertSpotter/Facebook/VirusTotal/Shodan/BeVigil kullanır, Wayback'ten son 5 yıl URL çeker.

---

## 13. Tek Bakışta: Web Recon Akışı

```
PASSIVE (önce — stealth)
├─ whois <domain>                          → sahip, registrar, NS, tarihler
├─ dig +short / MX / NS / TXT              → IP, mail, NS, SPF/verification
├─ crt.sh (cert transparency)              → subdomain listesi
├─ Google Dorks (site:, inurl:, filetype:) → login/config/backup/doc
├─ Wayback Machine                         → geçmiş asset/zafiyet
└─ theHarvester / Wappalyzer               → email, teknoloji

ACTIVE (sonra — hedefli, detection ↑)
├─ dnsenum -r (subdomain brute)            → gizli subdomain
├─ dig axfr @ns (zone transfer)            → tüm zone (misconfig'liyse)
├─ gobuster vhost --append-domain          → DNS'te OLMAYAN VHost
├─ wafw00f                                 → WAF var mı (önce!)
├─ curl -I (banner + redirect chain)       → server, framework, WordPress
├─ nikto -Tuning b                         → outdated software, header
├─ /robots.txt + /.well-known/             → gizli dizin, OIDC config
└─ crawl (Burp/ZAP spider)                 → link/comment/sensitive file
```

**Her web hedefi için checklist:**
```
☐ WHOIS — kayıt yaşı, registrar, NS (meşru mu, bulletproof mu)
☐ DNS kayıtları — A/MX/NS/TXT (TXT'de 3rd-party ipuçları)
☐ Subdomain — passive (crt.sh) + active (dnsenum) + VHost fuzz
☐ WAF tespiti ÖNCE (wafw00f) → tekniği adapte et
☐ Fingerprint — curl -I redirect chain sonuna kadar
☐ robots.txt + sitemap.xml + /.well-known/
☐ Google Dork — login/config/backup/exposed dosya
☐ Wayback — eski asset'ler (passive, stealth)
☐ Araç çıktısını context'le birleştir (tek veri değil, pattern)
```

---

## Kaynaklar

- **HTB Academy** — Information Gathering · Web Edition (CPTS path, Module 05)
- **Passive araçlar:** whois, dig/nslookup/host, crt.sh, Wayback Machine (archive.org), theHarvester, Wappalyzer/BuiltWith, Shodan
- **Active araçlar:** dnsenum/fierce/dnsrecon/amass, gobuster/feroxbuster/ffuf (vhost), wafw00f, nikto, whatweb
- **Automation:** FinalRecon, Recon-ng, SpiderFoot, OSINT Framework
- **Referanslar:** SecLists (wordlist), Google Hacking Database (GHDB), RFC 6962 (Cert Transparency), RFC 8615 (.well-known), RFC 8482 (ANY query)

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
