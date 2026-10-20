# CPTS-Study

**HTB CPTS** (Certified Penetration Testing Specialist) path'i için tuttuğum Türkçe çalışma notları.

Her modül kendi klasöründe bir `README.md` olarak duruyor. Notlar HTB Academy içeriğini **ezberden değil, anlayarak** özetleme amacıyla yazıldı; teknik terimler İngilizce bırakıldı, anlatım Türkçe. Tablolar, ASCII diyagramlar ve komut blokları ile görsel/aranabilir tutuldu.

> **Not stratejisi:** Her modül aynı iskeleti takip eder — kısa giriş + "Not stratejisi" + İçindekiler + bölümler + "Tek Bakışta Komut Referansı" + Kaynaklar. Amaç: sınavda/sahada takıldığında dönüp bakılacak hızlı referans.

---

## Modüller

| # | Modül | Konular | Durum |
|---|---|---|---|
| 01 | [Penetration Testing Process](01-penetration-testing-process/) | 8 aşamalı pentest süreci, pre-engagement/NDA/RoE, PoC, post-engagement | ✅ |
| 02 | [Getting Started](02-getting-started/) | Komut cheatsheet, Nibbles box uçtan uca, TTY upgrade | ✅ |
| 03 | [Network Enumeration with Nmap](03-network-enumeration-with-nmap/) | Scan tipleri, port state'leri, firewall/IDS evasion | ✅ |
| 04 | [Footprinting](04-footprinting/) | 13 servis (FTP/SMB/NFS/DNS/SMTP/SNMP/MySQL/MSSQL/Oracle...), port→komut haritası | ✅ |
| 05 | [Information Gathering — Web Edition](05-information-gathering-web-edition/) | WHOIS, DNS, subdomain/VHost fuzzing, fingerprinting, Google Dorking, Wayback | ✅ |
| 06 | [Vulnerability Assessment](06-vulnerability-assessment/) | VA vs pentest, CVSS, CVE/OVAL, Nessus/OpenVAS, raporlama | ✅ |
| 07 | [File Transfers](07-file-transfers/) | Windows/Linux × download/upload cheatsheet, LOLBAS/GTFOBins, evasion | ✅ |
| 08 | [Shells & Payloads](08-shells-and-payloads/) | Bind/reverse, msfvenom, Metasploit workflow, EternalBlue, web shell'ler | ✅ |
| 09 | [Using the Metasploit Framework](09-using-the-metasploit-framework/) | MSF derinleme: arama, DB/workspace, session/job, Meterpreter post-exploit, evasion | ✅ |

---

## Yapı

```
CPTS-Study/
├── README.md                              # bu dosya (indeks)
├── 01-penetration-testing-process/
│   └── README.md
├── 02-getting-started/
│   └── README.md
├── 03-network-enumeration-with-nmap/
│   └── README.md
├── 04-footprinting/
│   └── README.md
├── 05-information-gathering-web-edition/
│   └── README.md
├── 06-vulnerability-assessment/
│   └── README.md
├── 07-file-transfers/
│   └── README.md
├── 08-shells-and-payloads/
│   └── README.md
└── 09-using-the-metasploit-framework/
    └── README.md
```

---

## Kaynaklar

- **HTB Academy** — [CPTS / Penetration Tester Job Role Path](https://academy.hackthebox.com/path/preview/penetration-tester)
- **İlgili notlarım:** [CDSA-Study](https://github.com/demrcfurkan0/CDSA) (SOC Analyst path)

---

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.
