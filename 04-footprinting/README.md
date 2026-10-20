[← Tüm modüller](../README.md)

# 04 — Footprinting

HTB CPTS path'inin dördüncü modülü için tuttuğum çalışma notları. Modülün tamamı **servis enumeration** üzerine: her protokolün nasıl çalıştığı, default config'i, tehlikeli ayarları ve footprinting komutları. Kapsanan servisler: FTP/TFTP, SMB, NFS, DNS, SMTP, IMAP/POP3, SNMP, MySQL, MSSQL, Oracle TNS, IPMI, Linux remote mgmt (SSH/Rsync/R-services) ve Windows remote mgmt (RDP/WinRM/WMI). Ayrıca enumeration prensipleri + 6 katmanlı metodoloji + infrastructure/OSINT (crt.sh, Shodan, cloud storage).

> **Not stratejisi:** Bu modül devasa (HTB "2 gün" diyor) ve büyük bölümü senin zaten bildiğin şeyler. O yüzden her servisi **"footprinting komutları + tehlikeli ayarlar + neyi arıyorum"** şeklinde sıkıştırdım. Her servis için port, default config dosyası, enum komutları ve en kritik misconfiguration bir arada. Sınavda/sahada servis gördüğünde "hangi komut, hangi config, ne arıyorum" için dönüp bakılacak referans.

Notlar Türkçe, teknik terimler İngilizce bırakıldı.

---

## İçindekiler

**Bölüm I — Temeller**
1. [Enumeration Prensipleri & 6 Katmanlı Metodoloji](#1-enumeration-prensipleri--6-katmanlı-metodoloji)
2. [Infrastructure & OSINT (Katman 1-2)](#2-infrastructure--osint-katman-1-2)

**Bölüm II — Dosya & Paylaşım Servisleri**

3. [FTP / TFTP (21, 20 / 69 UDP)](#3-ftp--tftp)
4. [SMB (139, 445)](#4-smb)
5. [NFS (111, 2049)](#5-nfs)

**Bölüm III — Ağ Servisleri**

6. [DNS (53)](#6-dns)
7. [SMTP (25, 587, 465)](#7-smtp)
8. [IMAP / POP3 (143/993, 110/995)](#8-imap--pop3)
9. [SNMP (161/162 UDP)](#9-snmp)

**Bölüm IV — Veritabanları**

10. [MySQL (3306)](#10-mysql)
11. [MSSQL (1433)](#11-mssql)
12. [Oracle TNS (1521)](#12-oracle-tns)

**Bölüm V — Yönetim & Donanım**

13. [IPMI (623 UDP)](#13-ipmi)
14. [Linux Remote Mgmt — SSH / Rsync / R-services](#14-linux-remote-mgmt)
15. [Windows Remote Mgmt — RDP / WinRM / WMI](#15-windows-remote-mgmt)

16. [Tek Bakışta: Port → Komut Haritası](#16-tek-bakışta-port--komut-haritası)

---

# Bölüm I — Temeller

## 1. Enumeration Prensipleri & 6 Katmanlı Metodoloji

**Enumeration tanımı:** active (scan) + passive (3rd-party) bilgi toplama. OSINT ayrı bir süreç — sadece passive, enumeration'dan ayrı yürütülür. Enumeration bir **döngü**: topladığın bilgiye göre tekrar tekrar toplarsın.

> **Çekirdek felsefe:** "Amacımız sistemlere girmek değil, oraya giden **tüm yolları bulmak**." Yanlış yaklaşım: SSH/RDP/WinRM görünce hemen weak password brute-force — gürültülü, blacklist'lenirsin, defansı bilmeden saldırırsın. Doğru yaklaşım: altyapıyı anla, haritasını çıkar, sonra plan yap (hazine avcısı analojisi).

**3 prensip:**
```
1. Göründüğünden fazlası var. Tüm bakış açılarını değerlendir.
2. Gördüğün ile görmediğini ayır.
3. Her zaman daha fazla bilgi edinmenin yolu vardır. Hedefi anla.
```

**6 Katmanlı metodoloji** (duvarları aşmak gibi — her katmanda "giriş nerede" diye bakarsın):

| # | Katman | Ne aranır |
|---|---|---|
| 1 | **Internet Presence** | Domains, subdomains, vHosts, ASN, netblocks, IP, cloud instances |
| 2 | **Gateway** | Firewall, DMZ, IPS/IDS, EDR, proxy, NAC, segmentation, VPN, Cloudflare |
| 3 | **Accessible Services** | Service type, functionality, config, port, version, interface ← **bu modül burası** |
| 4 | **Processes** | PID, işlenen veri, task, source, destination |
| 5 | **Privileges** | Gruplar, kullanıcılar, izinler, kısıtlamalar, environment |
| 6 | **OS Setup** | OS type, patch level, network config, config dosyaları, hassas private dosyalar |

3 seviye: Infrastructure-based → Host-based → OS-based enumeration.

---

## 2. Infrastructure & OSINT (Katman 1-2)

Passive, 3rd-party üzerinden — "müşteri/ziyaretçi" gibi gizli kal.

### Subdomain & Domain keşfi
```bash
# crt.sh — Certificate Transparency logs (RFC 6962)
curl -s "https://crt.sh/?q=inlanefreight.com&output=json" | jq . \
  | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 \
  | awk '{gsub(/\\n/,"\n");}1;' | sort -u

# Company-hosted IP'leri ayıkla (3rd-party host'ları test ETME — izin yok)
for i in $(cat subdomainlist); do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4; done

# Shodan — internete açık cihaz/port/servis
for i in $(cat ip-addresses.txt); do shodan host $i; done

# DNS kayıtları
dig any inlanefreight.com
```

**DNS kayıtlarından okunan istihbarat** (TXT en değerlisi):
- **A** → IP↔(sub)domain · **MX** → mail server (Google ise skip) · **NS** → hosting provider ipucu
- **TXT** → 3rd-party verification key'leri → hangi servisler kullanılıyor: Atlassian (dev/collab), Google/Gmail (GDrive), LogMeIn (merkezi remote access — ele geçirilirse her şey), Mailgun (API test et: IDOR/SSRF), Outlook/O365 (OneDrive, Azure blob = SMB!), INWX (domain registrar, MS= değeri ≈ login ID)

### Cloud storage
Yanlış config'li → authsız erişim: S3 bucket (AWS), blob (Azure), cloud storage (GCP).
- **Google Dorks:** `inurl:`, `intext:` + şirket adı → PDF/doküman/kod
- **GrayHatWarfare** → AWS/Azure/GCP storage, dosya formatına göre filtrele (passive)
- **domain.glass** → altyapı + güvenlik durumu (Cloudflare "Safe" = gateway katmanı tespiti)
- Şirket adı kısaltmaları bucket adı olarak sık kullanılır
- **Leaked SSH keys** — overworked employee → private key sızar → passwordsuz login

---

# Bölüm II — Dosya & Paylaşım Servisleri

## 3. FTP / TFTP

**FTP (TCP 21 control + 20 data):** iki kanal. Active = client port açar (firewall sorun), Passive = server port bildirir (firewall-friendly). **Clear-text** — sniff'lenebilir.
**TFTP (UDP 69):** auth YOK, sadece global read/write dosyalar, sadece trusted network.

**Default config:** `/etc/vsftpd.conf` · deny listesi: `/etc/ftpusers`

**Tehlikeli ayarlar:**
```
anonymous_enable=YES       # anonymous login
anon_upload_enable=YES     # anonymous upload → web dizinine shell = RCE
anon_mkdir_write_enable=YES
write_enable=YES           # STOR,DELE,MKD...
```

**Footprinting:**
```bash
ftp 10.129.14.136          # Name: anonymous / (boş şifre)
#  ls -R  (ls_recurse_enable=YES ise tüm ağaç)
#  status / debug / trace  → server ayarları
#  get <file> / put <file> / !<local cmd>
wget -m --no-passive ftp://anonymous:anonymous@<ip>   # tüm dosyaları indir (gürültülü)

nmap -sV -p21 -sC -A <ip>  # ftp-anon, ftp-syst (STAT) NSE
nc -nv <ip> 21             # banner grab
openssl s_client -connect <ip>:21 -starttls ftp   # TLS → cert'ten hostname/email
```
> `hide_ids=YES` → UID/GID "ftp" olarak gizlenir. `ftp-anon` NSE anonymous'u teyit eder.

---

## 4. SMB

**Portlar:** NetBIOS 137/138/139 (eski, CIFS=SMBv1), **445** (modern, direct TCP). Samba = Linux/Unix implementasyonu. ACL tabanlı, share-level izinler.

| SMB Ver | Windows | Özellik |
|---|---|---|
| CIFS | NT 4.0 | NetBIOS |
| 1.0 | 2000 | direct TCP |
| 2.0/2.1 | Vista/7 | perf, signing |
| 3.0+ | 8/2012+ | multichannel, **E2E encryption**, AES |

**Default config:** `/etc/samba/smb.conf`

**Tehlikeli ayarlar:**
```
browseable = yes           # share'leri listele
read only = no / writable = yes
guest ok = yes             # passwordsuz
enable privileges = yes
create mask = 0777 / directory mask = 0777
logon script = x.sh / magic script = x.sh   # login/close'da script çalıştır
```

**Footprinting (çok araç = çapraz doğrulama şart):**
```bash
# Shares — null session (-N anonymous)
smbclient -N -L //10.129.14.128
smbclient //10.129.14.128/notes       # bağlan → ls, get, !<cmd>

nmap <ip> -sV -sC -p139,445           # smb2-security-mode, nbstat (Nmap az verir)

# rpcclient — MANUEL, en çok bilgi
rpcclient -U "" 10.129.14.128
#  srvinfo / enumdomains / querydominfo / netshareenumall
#  enumdomusers / queryuser <RID> / querygroup <RID>

# RID brute-force (restricted olsa bile queryuser çalışır)
for i in $(seq 500 1100); do rpcclient -N -U "" <ip> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|rid" && echo; done

samrdump.py <ip>                       # Impacket — user enum
smbmap -H <ip>                         # share + izinler
crackmapexec smb <ip> --shares -u '' -p ''
./enum4linux-ng.py <ip> -A             # hepsini otomatikleştirir (users, shares, policy, OS)
```
> **Neden önemli:** anonymous erişim → user keşfi → brute-force / password reuse. enum4linux-ng password policy'yi de verir (min_pw_length, lockout_threshold).

---

## 5. NFS

**Portlar:** RPC/portmapper **111** + **2049** (NFSv4 tek port). ONC-RPC/SUN-RPC tabanlı. Auth yok — RPC'ye devredilmiş (UNIX UID/GID). **UID/GID mapping client↔server aynı olmak zorunda değil** → sadece trusted network'te kullanılmalı.

| Ver | Özellik |
|---|---|
| v2/v3 | makine auth, UDP |
| v4 | Kerberos, firewall-friendly, ACL, stateful, port 2049 tek |

**Default config:** `/etc/exports`

**Tehlikeli ayarlar:**
```
rw                   # read+write
insecure             # >1024 portlar (normalde sadece root <1024)
nohide
no_root_squash       # root'un oluşturduğu dosyalar UID/GID 0 kalır ← KRİTİK
```
(`root_squash` default: root → anonymous'a map, güvenli)

**Footprinting:**
```bash
nmap <ip> -p111,2049 -sV -sC          # rpcinfo NSE
nmap --script nfs* <ip> -sV -p111,2049 # nfs-ls, nfs-showmount, nfs-statfs

showmount -e 10.129.14.128            # export listesi
mkdir target-NFS
sudo mount -t nfs <ip>:/ ./target-NFS/ -o nolock
ls -l mnt/nfs/   # username/group ile
ls -n mnt/nfs/   # UID/GID ile → local'de aynı UID oluşturup dosyalara eriş
sudo umount ./target-NFS
```
> **Escalation:** SSH erişimin varsa, NFS share'e hedef user'ın SUID'li shell'ini yükle, SSH user ile çalıştır → priv-esc. `no_root_squash` + writable → root shell.

---

# Bölüm III — Ağ Servisleri

## 6. DNS

**Port 53** (UDP sorgu, TCP zone transfer / >512 byte). Çoğunlukla unencrypted (DoT/DoH/DNSCrypt = şifreli çözümler).

**Record tipleri:**
| Record | Döner |
|---|---|
| A / AAAA | IPv4 / IPv6 |
| MX | mail server |
| NS | nameserver |
| TXT | SPF/DMARC/DKIM, verification key'ler |
| CNAME | alias |
| PTR | reverse (IP→domain) |
| SOA | zone yönetim bilgisi + admin email (`.`→`@`) |

**Default config (Bind9):** `named.conf.local/.options/.log` · zone: `/etc/bind/db.<domain>` · reverse: `/etc/bind/db.<subnet>`

**Tehlikeli ayarlar:**
```
allow-query { any; }
allow-recursion { any; }
allow-transfer { any; }     # ← KRİTİK: zone transfer herkese açık
zone-statistics
```

**Footprinting:**
```bash
dig soa <domain> @<ns>                 # admin email, zone bilgisi
dig ns inlanefreight.htb @<ns>         # başka nameserver'lar
dig CH TXT version.bind <ns>           # Bind versiyonu (varsa)
dig any inlanefreight.htb @<ns>        # tüm record'lar (hepsi gelmeyebilir)

# ZONE TRANSFER (AXFR) — allow-transfer any ise jackpot
dig axfr inlanefreight.htb @<ns>               # tüm iç hostlar + IP
dig axfr internal.inlanefreight.htb @<ns>      # iç zone → DC, VPN, WSUS...

# Subdomain brute-force (AXFR kapalıysa)
for sub in $(cat subdomains-top1million-110000.txt); do dig $sub.inlanefreight.htb @<ns> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done
dnsenum --dnsserver <ns> --enum -f subdomains.txt inlanefreight.htb
```
> **AXFR = altın madeni:** iç IP'ler, hostname'ler (dc1, dc2, vpn, ws1, wsus...) tek komutta dökülür.

---

## 7. SMTP

**Portlar:** 25 (default), 587 (STARTTLS auth'lu), 465 (SSL). Zincir: `MUA → MSA → MTA(relay) → MDA → mailbox`. ESMTP = TLS'li SMTP (EHLO sonrası STARTTLS).

**Default config:** `/etc/postfix/main.cf`

**Komutlar:** `AUTH PLAIN`, `HELO/EHLO`, `MAIL FROM`, `RCPT TO`, `DATA`, `RSET`, `VRFY` (user var mı), `EXPN`, `NOOP`, `QUIT`

**Tehlikeli ayar — Open Relay:**
```
mynetworks = 0.0.0.0/0     # ← herkes mail gönderebilir → spoofing
```

**Footprinting:**
```bash
telnet 10.129.14.128 25
#  EHLO x  → desteklenen komutlar
#  VRFY root  → 252 = user enum (config'e göre false-positive olabilir!)
#  MAIL FROM/RCPT TO/DATA → mail gönder (spoof test)

nmap <ip> -sC -sV -p25                          # smtp-commands NSE
nmap <ip> -p25 --script smtp-open-relay -v      # 16 test ile open relay tespiti
```
> `VRFY` güvenilmez — bazı server olmayan user'a da 252 döner. Araca körü körüne güvenme.

---

## 8. IMAP / POP3

**IMAP (143 / 993 SSL):** server'da online yönetim, klasör yapısı, çoklu client sync.
**POP3 (110 / 995 SSL):** sadece list/retrieve/delete, basit. Yüksek portlar (993/995) = TLS/SSL.

**Default config (Dovecot):** `dovecot-imapd` / `dovecot-pop3d`

| IMAP cmd | | POP3 cmd | |
|---|---|---|---|
| `LOGIN user pass` | giriş | `USER` / `PASS` | giriş |
| `LIST "" *` | dizinler | `STAT` / `LIST` | mail sayısı/boyut |
| `SELECT INBOX` | mailbox seç | `RETR id` | mail getir |
| `FETCH <id> all` | mail içeriği | `DELE id` | sil |
| `LOGOUT` | çık | `CAPA` / `QUIT` | özellikler/çık |

**Tehlikeli ayarlar:** `auth_debug`, `auth_debug_passwords`, `auth_verbose_passwords` (parolalar log'lanır), `auth_anonymous_username`

**Footprinting:**
```bash
nmap <ip> -sV -p110,143,993,995 -sC   # ssl-cert (CN=hostname, org, email), capabilities

curl -k 'imaps://<ip>' --user user:pass        # giriş + LIST
curl -k 'imaps://<ip>' --user user:pass -v     # TLS detay + banner + versiyon

openssl s_client -connect <ip>:imaps           # manuel TLS etkileşim
openssl s_client -connect <ip>:pop3s
```

---

## 9. SNMP

**Portlar:** 161 UDP (query/command), 162 UDP (traps). Router/switch/server/IoT yönetimi.
- **MIB:** queryable objelerin ASN.1 tree'si · **OID:** tree'deki node, dot notation
- **v1/v2c:** community string plain-text, encryption YOK · **v3:** user+pass auth + encryption

**Community string = parola** (ro/rw). Default: `public`/`private`.

**Default config:** `/etc/snmp/snmpd.conf`

**Tehlikeli ayarlar:**
```
rwuser noauth                          # authsız tüm OID tree
rwcommunity <string> <IPv4>            # her yerden tam erişim
rwcommunity6 <string> <IPv6>
```

**Footprinting:**
```bash
snmpwalk -v2c -c public 10.129.14.128              # tüm OID'leri dök (sysDescr, user, paketler...)
snmpwalk -v2c -c public <ip> 1.3.6.1.2.1.1.5.0     # tek OID (hostname)

onesixtyone -c snmp.txt <ip>           # community string brute (SecLists)
braa public@<ip>:.1.3.6.*              # bilinen string ile OID brute
```
> snmpwalk çıktısında: OS versiyonu, admin email, kurulu paketler (versiyonlarıyla → CVE), process'ler, command-line cred'ler sızabilir.

---

# Bölüm IV — Veritabanları

## 10. MySQL

**Port 3306.** Open-source RDBMS. MariaDB = MySQL fork'u (Oracle satın alınca). LAMP/LEMP stack. WordPress vb. CMS'ler user/pass burada tutar.

**Default config:** `/etc/mysql/mysql.conf.d/mysqld.cnf`

**Tehlikeli ayarlar (plain-text!):** `user`, `password`, `admin_address`, `debug`, `sql_warnings`, `secure_file_priv`

**Footprinting:**
```bash
nmap <ip> -sV -sC -p3306 --script mysql*    # mysql-info, mysql-empty-password, mysql-users
#  ⚠ mysql-empty-password sık FALSE-POSITIVE → elle teyit et:
mysql -u root -h <ip>                        # şifresiz dene
mysql -u root -pP4SSw0rd -h <ip>             # -p ile şifre ARALARINDA BOŞLUK YOK

# İçeride:
show databases;              # information_schema, mysql, performance_schema, sys
use <db>; show tables;
show columns from <table>;
select * from <table>;
select version();
```
> Önemli DB'ler: `sys` + `information_schema` (metadata). Çıktıları her zaman elle doğrula.

---

## 11. MSSQL

**Port 1433** (+ 1434 UDP DAC, named pipe). Microsoft, closed-source, .NET native. Çoğunlukla Windows'ta.

**Default DB'ler:** `master` (sistem bilgisi), `model` (template), `msdb` (agent/job), `tempdb` (geçici), `resource` (read-only sistem objeleri).

**Default:** `NT SERVICE\MSSQLSERVER`, Windows Authentication, encryption ENFORCE EDİLMEZ.

**Tehlikeli ayarlar:** şifresiz client bağlantısı, self-signed cert (spoof'lanabilir), named pipes, **zayıf/default `sa` cred** (admin disable etmeyi unutur).

**Footprinting:**
```bash
# Nmap (default sa, boş şifre dene)
nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes \
  --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <ip>

# Metasploit
msf> use auxiliary/scanner/mssql/mssql_ping   # ServerName, InstanceName, Version, port, pipe

# Bağlan (Impacket)
locate mssqlclient
python3 mssqlclient.py Administrator@<ip> -windows-auth
SQL> select name from sys.databases
```
> Diğer client'lar: mssql-cli, SQL Server PowerShell, HeidiSQL, SSMS (saved cred'ler için değerli).

---

## 12. Oracle TNS

**Port 1521** (Transparent Network Substrate). Oracle DB ↔ app iletişimi. Oracle 8i/9i remote mgmt açık, 10g/11g değil.

**Config:** `tnsnames.ora` (client-side, service→adres) + `listener.ora` (server-side, listener) → `$ORACLE_HOME/network/admin`

**Default cred'ler (not al!):** Oracle 9 → `CHANGE_ON_INSTALL` · Oracle 10 → şifresiz · DBSNMP servisi → `dbsnmp` · ODAT'ın bulduğu klasik: `scott/tiger`

**SID** = DB instance unique adı (bağlanmak için gerekli).

**Footprinting (önce ODAT kur — section'da pip komutları var):**
```bash
nmap -p1521 -sV <ip> --open                              # TNS listener versiyon
nmap -p1521 -sV <ip> --open --script oracle-sid-brute    # SID brute (örn: XE)

./odat.py all -s <ip>                                     # tüm modüller → cred bulur (scott/tiger)

# sqlplus ile bağlan
sqlplus scott/tiger@<ip>/XE
sqlplus scott/tiger@<ip>/XE as sysdba                     # yüksek yetki dene
SQL> select table_name from all_tables;
SQL> select * from user_role_privs;
SQL> select name, password from sys.user$;                # hash dump → offline crack

# Web shell upload (webroot biliniyorsa)
echo "test" > testing.txt
./odat.py utlfile -s <ip> -d XE -U scott -P tiger --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
curl http://<ip>/testing.txt
```
Webroot default: Linux `/var/www/html`, Windows `C:\inetpub\wwwroot`.

---

# Bölüm V — Yönetim & Donanım

## 13. IPMI

**Port 623 UDP.** Hardware-based host mgmt — BIOS/CPU/OS'tan bağımsız çalışır, host kapalıyken bile (güç+LAN yeter). BMC'ler (Baseboard Management Controller) üzerinden. **BMC erişimi ≈ fiziksel erişim** (reboot, power off, OS reinstall).

Yaygın BMC: HP iLO, Dell DRAC, Supermicro IPMI (hepsi web console + SSH/Telnet + 623 UDP).

**Footprinting:**
```bash
nmap -sU --script ipmi-version -p 623 <host>
msf> use auxiliary/scanner/ipmi/ipmi_version
```

**Default cred'ler:**
| Ürün | User | Pass |
|---|---|---|
| Dell iDRAC | root | calvin |
| HP iLO | Administrator | 8-char random (rakam+büyük harf) |
| Supermicro IPMI | ADMIN | ADMIN |

**Tehlikeli — RAKP protokol açığı (IPMI 2.0):** auth öncesi server, **herhangi bir geçerli user'ın** salted SHA1/MD5 hash'ini gönderir → offline crack (Hashcat mode **7300**). Spec'in parçası, direkt "fix" yok.
```bash
msf> use auxiliary/scanner/ipmi/ipmi_dumphashes    # hash dump + CRACK_COMMON
# HP iLO factory default crack:
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```
> Internal pentest playbook'unda olmalı. Crack'lenen IPMI parolası sık sık diğer sistemlerde reuse edilir → root SSH.

---

## 14. Linux Remote Mgmt

### SSH (TCP 22)
OpenSSH (OpenBSD fork). SSH-2 >> SSH-1 (SSH-1 MITM'e açık). 6 auth metodu (password, **public-key**, host-based, keyboard, challenge-response, GSSAPI).

**Config:** `/etc/ssh/sshd_config`

**Tehlikeli ayarlar:**
```
PasswordAuthentication yes    # brute-force'a açık
PermitEmptyPasswords yes
PermitRootLogin yes
Protocol 1                    # eski encryption
X11Forwarding yes             # 7.2p1'de command injection (2016)
AllowTcpForwarding yes / PermitTunnel
```

**Footprinting:**
```bash
./ssh-audit.py <ip>                          # banner, algoritmalar, zayıf olanlar [fail]
ssh -v user@<ip>                             # "Authentications that can continue" → metodlar
ssh -v user@<ip> -o PreferredAuthentications=password   # brute için metod sabitle
```
> Banner okuma: `SSH-1.99-...` = hem v1 hem v2; `SSH-2.0-OpenSSH_8.2p1` = sadece v2.

### Rsync (TCP 873)
Delta-transfer ile dosya kopyalama (backup/mirror). Authsız share listelenebilir.
```bash
nmap -sV -p 873 <ip>                         # protocol 31
nc -nv <ip> 873                              # #list → share'ler
rsync -av --list-only rsync://<ip>/dev       # share içeriği (secrets.yaml, .ssh...)
rsync -av rsync://<ip>/dev                   # indir
rsync -av -e ssh rsync://<ip>/dev            # SSH üzerinden
```

### R-Services (512 rexec, 513 rlogin, 514 rsh/rcp)
SSH öncesi Unix remote access. **Unencrypted** → MITM. Solaris/HP-UX/AIX. `rcp/rsh/rexec/rlogin/rwho/rusers`.

**Access control açığı:** `/etc/hosts.equiv` (global) + `.rhosts` (per-user) trusted host listesi — PAM'i bypass eder.
```
htb-student 10.0.17.5
+ 10.0.17.10          # + = wildcard → KRİTİK misconfiguration
+ +                   # herkese her şey
```
```bash
nmap -sV -p 512,513,514 <ip>
rlogin <ip> -l htb-student           # .rhosts misconfig'i varsa credsiz giriş
rwho                                 # ağdaki aktif oturumlar (513 UDP)
rusers -al <ip>                      # detaylı: user, host, TTY, login zamanı
```

---

## 15. Windows Remote Mgmt

### RDP (TCP/UDP 3389)
GUI remote access. Server'da default kurulu, NLA (Network Level Auth) ile. TLS/SSL ama cert self-signed default.

**Footprinting:**
```bash
nmap -sV -sC <ip> -p3389 --script rdp*       # rdp-enum-encryption (NLA?), rdp-ntlm-info (hostname, domain, build)
# ⚠ Nmap RDP cookie "mstshash=nmap" → EDR yakalayabilir, lockout riski
./rdp-sec-check.pl <ip>                       # Cisco — protokol/encryption desteği (authsız)

xfreerdp /u:user /p:"pass" /v:<ip>           # bağlan (rdesktop/Remmina de olur)
```

### WinRM (TCP 5985 HTTP / 5986 HTTPS)
SOAP tabanlı, Server 2012+ default açık. WinRS = komut çalıştırma. PS remoting + event log merge için gerekli.
```bash
nmap -sV -sC <ip> -p5985,5986 --disable-arp-ping -n   # Microsoft-HTTPAPI/2.0
# PowerShell: Test-WsMan <host>
evil-winrm -i <ip> -u Cry0l1t3 -p 'P455w0rD!'         # cred ile shell
```

### WMI (TCP 135 → random port)
CIM/WBEM implementasyonu. Windows'un en kritik yönetim arayüzü (neredeyse tüm ayarlara read/write). PowerShell/VBScript/WMIC üzerinden.
```bash
wmiexec.py Cry0l1t3:'P455w0rD!'@<ip> "hostname"       # Impacket
```

---

## 16. Tek Bakışta: Port → Komut Haritası

| Port | Servis | İlk komut | Config dosyası |
|---|---|---|---|
| 21/20 | FTP | `ftp <ip>` (anonymous) · `nmap --script ftp-anon` | `/etc/vsftpd.conf`, `/etc/ftpusers` |
| 69/udp | TFTP | `tftp` get/put | — |
| 53 | DNS | `dig axfr <domain> @<ns>` | `named.conf`, `db.<domain>` |
| 110/995 | POP3 | `nmap -sC -p110,995` | dovecot |
| 111/2049 | NFS | `showmount -e <ip>` → `mount -t nfs` | `/etc/exports` |
| 135 | WMI | `wmiexec.py user:pass@<ip> cmd` | — |
| 139/445 | SMB | `smbclient -N -L //<ip>` · `rpcclient -U "" <ip>` · `enum4linux-ng -A` | `/etc/samba/smb.conf` |
| 143/993 | IMAP | `curl -k imaps://<ip> --user u:p` | dovecot |
| 161/udp | SNMP | `snmpwalk -v2c -c public <ip>` · `onesixtyone` | `/etc/snmp/snmpd.conf` |
| 22 | SSH | `ssh-audit.py <ip>` · `ssh -v` | `/etc/ssh/sshd_config` |
| 25/587/465 | SMTP | `telnet <ip> 25` (VRFY) · `--script smtp-open-relay` | `/etc/postfix/main.cf` |
| 512/513/514 | R-services | `rlogin <ip> -l user` · `rusers -al` | `.rhosts`, `/etc/hosts.equiv` |
| 623/udp | IPMI | `nmap -sU --script ipmi-version -p623` · `ipmi_dumphashes` (hashcat -m 7300) | — |
| 873 | Rsync | `rsync -av --list-only rsync://<ip>/<share>` | — |
| 1433 | MSSQL | `mssqlclient.py Admin@<ip> -windows-auth` · `mssql_ping` | — |
| 1521 | Oracle TNS | `nmap --script oracle-sid-brute` · `odat.py all -s <ip>` · `sqlplus` | `tnsnames.ora`, `listener.ora` |
| 3306 | MySQL | `mysql -u root -p<pass> -h <ip>` | `mysqld.cnf` |
| 3389 | RDP | `nmap --script rdp*` · `xfreerdp /u: /p: /v:` | — |
| 5985/5986 | WinRM | `evil-winrm -i <ip> -u u -p p` | — |

**Her servis için ortak checklist:**
```
☐ Default/anonymous erişim var mı? (FTP anon, SMB null, NFS mount, SNMP public)
☐ Default cred'ler? (sa, scott/tiger, ADMIN/ADMIN, root/calvin)
☐ Unencrypted mi? (FTP, R-services, SNMPv1/2, SMTP, IMAP/POP3 düz port)
☐ Version → CVE var mı?
☐ Config dosyası okunabiliyor mu? (plain-text cred)
☐ Password reuse — bir yerde bulunan cred her serviste denenir
☐ Araç çıktısını ELLE doğrula (VRFY, mysql-empty-password false-positive verir)
```

---

## Kaynaklar

- **HTB Academy** — Footprinting (CPTS path, Module 04)
- **Araçlar:** nmap+NSE, rpcclient, enum4linux-ng, smbmap, crackmapexec, Impacket (samrdump/mssqlclient/wmiexec), dig/dnsenum, snmpwalk/onesixtyone/braa, ODAT, sqlplus, ssh-audit, rsync, evil-winrm, xfreerdp, Metasploit (ipmi/mssql modülleri)
- **OSINT:** crt.sh, Shodan, GrayHatWarfare, domain.glass, Google Dorks
- **Referanslar:** SecLists (wordlist), Hashcat (IPMI mode 7300), RFC 6962 (Cert Transparency)

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
