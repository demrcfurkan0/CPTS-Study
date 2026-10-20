[← Tüm modüller](../README.md)

# 01 — Penetration Testing Process

HTB CPTS path'inin ilk modülü için tuttuğum çalışma notları. Bir penetration test'in uçtan uca süreci: contract ve scoping'den (pre-engagement), information gathering → vulnerability assessment → exploitation → post-exploitation → lateral movement zincirine, oradan PoC, raporlama ve post-engagement kapanışına kadar. Bu modül teknik değil, **metodoloji** modülü — path'in geri kalanındaki her tekniğin hangi aşamaya oturduğunu gösteren harita.

Notlar Türkçe yazıldı, teknik terimler bilinçli olarak İngilizce bırakıldı — saha dili bu şekilde ve terimleri çevirmek aramayı zorlaştırıyor.

---

## İçindekiler

**Bölüm I — Temeller**
1. [Pentest Nedir, Ne Değildir](#1-pentest-nedir-ne-değildir)
2. [Risk Management](#2-risk-management)
3. [Testing Methods — External vs Internal](#3-testing-methods--external-vs-internal)
4. [Pentest Türleri — Box Rengi](#4-pentest-türleri--box-rengi)
5. [Laws & Regulations](#5-laws--regulations)

**Bölüm II — Süreç Haritası**

6. [8 Aşamalı Süreç Mantığı](#6-8-aşamalı-süreç-mantığı)

**Bölüm III — Aşamalar Tek Tek**

7. [Pre-Engagement](#7-pre-engagement)
8. [Information Gathering](#8-information-gathering)
9. [Vulnerability Assessment](#9-vulnerability-assessment)
10. [Exploitation](#10-exploitation)
11. [Post-Exploitation](#11-post-exploitation)
12. [Lateral Movement](#12-lateral-movement)
13. [Proof-of-Concept](#13-proof-of-concept)
14. [Post-Engagement](#14-post-engagement)

**Bölüm IV — Kapanış**

15. [Pratik Metodolojisi](#15-pratik-metodolojisi)
16. [Tek Bakışta Özet](#16-tek-bakışta-özet)

---

# Bölüm I — Temeller

## 1. Pentest Nedir, Ne Değildir

| Terim | Tanım | Ayırt edici nokta |
|---|---|---|
| **Penetration Test** | IT altyapısına karşı organize, hedefli ve **authorized** saldırı girişimi; gerçek saldırganların yöntemleriyle zafiyete açıklığı ölçer | Yetki şart — yazılı izin olmadan aynı aksiyonlar suç |
| **Pentest hedefi** | İncelenen sistemlerdeki **TÜM** zafiyetleri bulmak ve güvenliği artırmak | "Kapsamlı" — tek yol bulup durmaz |
| **Red Team Assessment** | Senaryo bazlı; sadece belirli bir **end goal**'e ulaşmak için gereken zafiyetlere odaklanır (CEO mailine erişmek, kritik sunucudaki flag) | Dar ve hedefe kilitli — kapsamlı değil |
| **Vulnerability Assessment** | Saf **otomatik** araçlarla (Nessus, Qualys, OpenVAS) bilinen zafiyetlere karşı tarama | İnsan yok, konfigürasyona adapte olamaz |

**Kritik ayrım — Pentest ≠ Vulnerability Scan:**
- VA = tamamen otomatik, bilinen issue'lara karşı, hedefe uyarlanamaz.
- Pentest = otomatik + **manuel** doğrulama; büyük ölçüde manuel information gathering sonrası; her sisteme **bireysel olarak uyarlanır**.

**En çok sınavda/işte tökezlenen cümle:**

> A pentest is **not monitoring** — it is a **momentary snapshot** of security status.

Yani rapora "bu bir anlık görüntüdür, sürekli izleme değildir" ibaresi konur. Sistem düzeltmek **müşterinin** sorumluluğu; biz trusted advisor olarak zafiyeti + reproduction + remediation öneririz, **patch uygulamayız**.

**Yetki & kapsam uyarıları:**
- Müşteri sadece **kendi** asset'lerini test ettirebilir. Üçüncü taraf (cloud, hosting) varsa onlardan da yazılı izin gerekir.
- AWS gibi bazı sağlayıcılar belirli servisler için ön-izin istemez — ama bunu her zaman scoping'de teyit et.
- Çalışanlara genelde haber verilmez; ama yönetim bilgilendirmeyi seçebilir (privacy expectation'ı yoksa bilme hakları var).
- Bulunan kişisel veri (isim, maaş, kredi kartı) gizli tutulur — Data Protection Act. Öneri: parolaları değiştirin, DB'yi encrypt edin.

---

## 2. Risk Management

Pentest, şirketin risk management sürecinin bir parçası. Amaç: confidentiality, integrity, availability'ye (CIA) zarar verebilecek riskleri identify → evaluate → mitigate ederek riski **kabul edilebilir** seviyeye indirmek.

**Her risk elimine edilemez** — residual/inherent risk her zaman kalır.

| Risk stratejisi | Nasıl | Örnek |
|---|---|---|
| **Accept** | Riski olduğu gibi kabul et | Düşük etkili, düşük olasılıklı |
| **Transfer** | Başka tarafa aktar | Sigorta, 3rd-party service provider'a contract ile |
| **Avoid** | Riski doğuran aktiviteden kaçın | O servisi hiç çalıştırmamak |
| **Mitigate** | Olasılığı/etkiyi azalt | Security control + policy uygula |

**Inherent risk:** uygun kontroller yerindeyken bile var olan risk seviyesi. Sıfırlanamaz.

---

## 3. Testing Methods — External vs Internal

Her pentest iki perspektiften birinden başlar. **Başlangıç noktası** metodun en kritik kararı.

| | External Penetration Test | Internal Penetration Test |
|---|---|---|
| Konum | İnternetten, anonim kullanıcı gibi | Kurum ağının **içinden** |
| Nereden | Kendi host'umuz (VPN ile, ISP ban yememek için) veya VPS | Dış pentest başarısı sonrası **veya** assumed breach |
| Hedef | External-facing host'lara erişim, sensitive data, iç ağa giriş | İç ağda yayılma; izole/internetsiz sistemler → **fiziksel varlık** gerekebilir |
| Odak | Perimeter savunması | İç segmentasyon, privilege, yanal hareket |

**Assumed breach:** "Saldırgan zaten içeri girdi" varsayımıyla başlama. External aşamayı atlayıp doğrudan iç ağ testine geçmek için kullanılır — zaman tasarrufu.

**Stealth seçenekleri** (external'da sık istenir):
- **Non-evasive** — gürültülü, maksimum derinlik
- **Hybrid** — sessiz başla, kademeli "gürültülü"leş; blue team hangi seviyede fark ediyor ölç
- **Fully evasive** — firewall ban / IDS-IPS / alarm tetiklemeden

---

## 4. Pentest Türleri — Box Rengi

Tür, bize **ne kadar bilgi verildiğini** belirler. Az bilgi = daha uzun, daha karmaşık.

| Tür | Verilen bilgi | Ne demek |
|---|---|---|
| **Blackbox** | Minimum — sadece IP ve domain | Sıfırdan recon; en uzun süre (özellikle tüm network'te) |
| **Greybox** | Genişletilmiş — URL, hostname, subnet | Kısmi iç görü |
| **Whitebox** | Maksimum — her şey açık (config, admin creds, source code) | İç yapının tam görünümü; internal bilgiyle hazırlık |
| **Red-Teaming** | Değişken — fiziksel + social engineering dahil olabilir | Yukarıdakilerden biriyle kombine edilebilir |
| **Purple-Teaming** | Değişken — defender ile **yakın çalışma** odaklı | Blue team ile birlikte, bilgi paylaşımlı |

**Test edilebilecek ortamlar** (sıklıkla karışık): Network, Web App, Mobile, API, Thick Clients, IoT, Cloud, Source Code, Physical Security, Employees, Hosts, Server Security, Policies, Firewalls, IDS/IPS.

---

## 5. Laws & Regulations

Her ülkenin bilgisayar suçları, veri koruma ve interception'a dair kendi yasaları var. Pentester bunları bilmek zorunda — ihlali civil/criminal ceza.

| Kategori | USA | Europe | UK |
|---|---|---|---|
| Kritik altyapı + kişisel veri koruma | CISA | **GDPR** | Data Protection Act 2018 |
| Yetkisiz erişimi suç sayan | **CFAA** | NISD 2 | **Computer Misuse Act 1990** |
| Copyright koruma atlatmayı yasaklayan | **DMCA** | Cybercrime Convention | — |
| Elektronik iletişim interception | ECPA | E-Privacy Directive | RIPA 2000 / IPA 2016 |
| Sağlık verisi (PHI) | **HIPAA** | — | — |
| Çocuk verisi (<13 yaş) | **COPPA** | — | — |

**Akılda kalması gerekenler:**
- **CFAA (USA):** yetkisiz bilgisayar erişimini suç sayar; tanımları "fazla geniş" diye eleştirilir — meşru güvenlik araştırmasını bile kapsayabileceği endişesi.
- **GDPR (Europe):** EU vatandaşının verisini işleyen **her** şirkete uygulanır (şirketin konumu fark etmez). Ceza: global yıllık cironun %4'ü **veya** 20M€ — hangisi yüksekse.
- **Computer Misuse Act (UK):** yetkisiz erişim/değişiklik/fraud suç. Pentest'te yazılı consent yoksa **tam da bu yasayı** çiğnersin.

**Precautionary Measures (her testte uygula):**

```
☐ Sistem/network sahibinden YAZILI consent al
☐ Sadece consent kapsamında test et, limitlere uy
☐ Sistemlere zarar vermeyecek önlemler al
☐ İzinsiz kişisel veri erişme / kullanma / ifşa etme
☐ Taraflardan birinin onayı olmadan iletişim intercept etme
☐ HIPAA kapsamındaki sistemleri uygun yetki olmadan test etme
```

---

# Bölüm II — Süreç Haritası

## 6. 8 Aşamalı Süreç Mantığı

**Tanım:** Penetration testing process = pentester'ın önceden tanımlı hedefe giden bir yol bulmak için yaptığı **ardışık adımlar ve event'ler** dizisi.

**En kritik kavramsal nokta:** Bu bir **step-by-step reçete değil**. Süreç "coarse and flexible" (kaba ve esnek) olmalı — çünkü her müşterinin altyapısı, isteği ve beklentisi farklı. Deterministic bir süreçtir: her adım, keşfettiğimiz/provoke ettiğimiz event ve sonuçlara dayanır.

```
┌───────────────┐   ┌─────────────────────┐   ┌────────────────────────┐   ┌──────────────┐
│ Pre-Engagement│──►│ Information Gathering│──►│ Vulnerability          │──►│ Exploitation │──┐
│ (contract)    │   │                      │◄──┤ Assessment             │   │              │  │
└───────────────┘   └──────────▲───────────┘   └────────────────────────┘   └──────────────┘  │
                               │  (sürekli geri dönüş)                                         │
        ┌──────────────────────┴───────────────────────────────────────────────────┐          │
        │                                                                            ▼          ▼
┌──────────────┐   ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────────────────┐
│ Post-        │◄──┤ Lateral         │◄──┤ Proof-of-Concept │   │ Post-Exploitation                │
│ Engagement   │   │ Movement        │   │ (dokümantasyon)  │◄──┤ (priv-esc, pillaging, persistence)│
│ (rapor,temiz)│   │ (iteratif döngü)│   └──────────────────┘   └──────────────────────────────────┘
└──────────────┘   └─────────────────┘
```

Oklar tek yönlü değil: Information Gathering ↔ Vulnerability Assessment arası sürekli gidip gelinir; Lateral Movement içinde Post-Exploitation adımları tekrar tekrar koşturulur.

| # | Aşama | Tek cümlelik özü |
|---|---|---|
| 1 | **Pre-Engagement** | Müşteriyi eğit, contract'ı ayarla, scope'u ve kuralları yazılı bağla |
| 2 | **Information Gathering** | Şirket, çalışan ve altyapı hakkında mümkün olan her bilgiyi topla |
| 3 | **Vulnerability Assessment** | Toplanan bilgiyi analiz et, bilinen zafiyet/attack vector'leri çıkar |
| 4 | **Exploitation** | Zafiyetleri hedefe uyarla, initial access için çalıştır |
| 5 | **Post-Exploitation** | İçeriden bilgi topla, privilege escalate et, pillage et, persistence kur |
| 6 | **Lateral Movement** | İç ağda yatay ilerle, başka host'lara eş/yüksek yetkiyle eriş |
| 7 | **Proof-of-Concept** | Zafiyetin var olduğunu adım adım kanıtla, mümkünse otomatikleştir |
| 8 | **Post-Engagement** | Raporla, temizle, walkthrough yap, veriyi arşivle/imha et |

**Pillaging nerede?** Ayrı bir aşama **değil** — Information Gathering ve Privilege Escalation'ın içinde, hedef sistemde lokal olarak yapılan ayrılmaz parça. HTB'nin ayrı modülü yok, bilinçli bir tercih.

---

# Bölüm III — Aşamalar Tek Tek

## 7. Pre-Engagement

Asıl testten önceki hazırlık. **Üç ana bileşen:**

```
1. Scoping Questionnaire  →  2. Pre-Engagement Meeting  →  3. Kick-Off Meeting
   (hizmeti netleştir)        (contract detayları)          (herkes imzaladı, başla)
```

Hepsinden önce bir **NDA** imzalanmalı.

### NDA Türleri

| Tür | Kim gizli tutar |
|---|---|
| **Unilateral** | Tek taraf; diğeri bilgiyi 3. tarafla paylaşabilir |
| **Bilateral** | **Her iki taraf** — pentester'ı koruyan en yaygın tür |
| **Multilateral** | İkiden fazla taraf (cooperative network testinde) |

### Dökümanlar ve Ne Zaman Oluşturulur

| # | Döküman | Zamanlama |
|---|---|---|
| 1 | **NDA** | İlk temas sonrası |
| 2 | **Scoping Questionnaire** | Pre-engagement meeting'den önce |
| 3 | **Scoping Document** | Pre-engagement meeting sırasında |
| 4 | **Proposal (Contract / SoW)** | Pre-engagement meeting sırasında |
| 5 | **Rules of Engagement (RoE)** | Kick-off meeting'den önce |
| 6 | **Contractors Agreement** (fiziksel test) | Kick-off meeting'den önce |
| 7 | **Reports** | Test sırasında ve sonrasında |

> Tüm bu dökümanlar hazırlandıktan sonra bir **avukat** tarafından gözden geçirilmeli.

### Kim Sözleşme İmzalayabilir?
Herkesten iş alınmaz — yetkisiz bir çalışan kötü niyetle bizi tutabilir. Yetkili olabilecekler: CEO, CTO, CISO, CSO, CRO, CIO, VP of Internal Audit, Audit Manager, VP/Director of IT/InfoSec. Büyük kurumlarda C-level yerine IT/Audit/Security senior management.

### Scoping Questionnaire — Kritik Sorular
Assessment türü seçimi (Internal/External VA, Internal/External Pentest, Wireless, App Security, Physical, Social Engineering, Red Team, Web App) + şu sayısal bilgiler:
- Kaç live host? Kaç IP/CIDR range? Kaç domain/subdomain?
- Kaç wireless SSID? Kaç web/mobile app (auth varsa kaç rol)?
- Phishing: kaç kullanıcı, listeyi müşteri mi verecek yoksa OSINT mi?
- Physical: kaç lokasyon, coğrafi dağınık mı?
- NAC bypass gerekli mi? Anonim kullanıcı mı, domain user mı?
- Information disclosure: black/grey/white box? Evasiveness: non/hybrid/fully evasive?

Bu bilgi **doğru kaynak atama + doğru teklif (timeline + maliyet)** için şart. 10 IP'lik external ≠ 30 adet /24'lük internal.

### RoE — Checklist (özet)
Introduction, Contractor, Pentesters, Contact Info, Purpose, Goals, **Scope** (tüm IP/domain/URL/CIDR), Lines of Communication, Time Estimation, Time of Day, Pentest Type, Locations, **Methodologies** (OSSTMM, PTES, OWASP), Objectives/Flags, Evidence Handling, System Backups, Information Handling, Incident Handling & Reporting, Status Meetings, Reporting, Retesting, Disclaimers & Limitation of Liability, **Permission to Test**.

### Kick-Off Meeting — Söylenecekler
- Genelde **DoS testi yok**.
- Kritik zafiyet bulunursa (örn. unauthenticated RCE, SQLi) → test **durur**, vulnerability notification report hazırlanır, emergency contact aranır (çoğunlukla external'da).
- Internal test sadece şu durumlarda durdurulur: sistem yanıt vermez olursa, illegal içerik bulunursa, veya ağda **mevcut bir external threat actor / önceki breach** tespit edilirse.
- Riskler anlatılır: çok log/alarm üretilebilir; brute force bazı kullanıcıları **kilitleyebilir**.
- Anlatım en teknik-olmayan dinleyiciye göre ayarlanır (CTO/CISO dışındakiler için "magic" gibi görünür).

### Contractors Agreement
Sadece **fiziksel test** varsa gerekir. Fiziksel izinde farklı yasalar geçerli; çalışanlar haberdar olmayabilir, yakalanırsak polis çağrılabilir. Bu belge **"get out of jail free card"**. İçerik: adres, bina adı, kat, oda ID'leri, fiziksel bileşenler, timeline, **notarization**, permission to test.

---

## 8. Information Gathering

Pentest'in **temel taşı** ve en sık dönülen aşama. Tüm exploitation adımları buradan toplanan bilgiye dayanır. Dört kategori — **hepsi her testte yapılmalı:**

| Kategori | Ne yapılır |
|---|---|
| **Open-Source Intelligence (OSINT)** | Kamuya açık kaynaklardan (social media, job postings, GitHub, StackOverflow) bilgi |
| **Infrastructure Enumeration** | DNS ile sunucu/host haritası; name/mail/web server, cloud instance; güvenlik önlemlerini (WAF, firewall) tespit |
| **Service Enumeration** | Servisleri, **versiyonlarını** belirle; eski versiyon = bilinen zafiyet |
| **Host Enumeration** | Her host'un OS'i, servisleri, versiyonları, rolü, hangi portta ne var |

**OSINT'in gücü ve riski:**
- Dakikalar içinde password, hash, key, token bulunabilir — yanlış yapılandırılmış GitHub repoları, StackOverflow'a yapıştırılmış kod blokları.
- Böyle kritik bir bulgu testin başında çıkarsa → RoE'deki **Incident Handling & Report** prosedürüne göre bildir; müşteri admin'i proceed etmeden önce incelesin.

**Pillaging (Information Gathering'in alt parçası):**
Post-exploitation'a geçince, ele geçirilmiş host üzerinde **lokal** hassas bilgi toplama — çalışan isimleri, müşteri verisi, parolalar (share'lerde, script'lerde, config'lerde, vault'larda, Excel/Word/.txt'lerde, mail'de). Amaç: saldırının etkisini göstermek + lateral movement için input (parola vb.) bulmak.

> **İç perspektif farkı:** Admin'ler internetten erişilemeyen servisleri "güvenli" sanıp ihmal eder — misconfiguration'ların çoğu burada çıkar.

---

## 9. Vulnerability Assessment

Information Gathering çıktısını **analiz** etme aşaması. Analiz = bir olayın/sürecin köken ve etkisinin detaylı incelenmesi.

### Dört Analiz Türü

| Tür | Ne yapar | Zaman yönü |
|---|---|---|
| **Descriptive** | Veri setini özellikleriyle tanımlar; hata/outlier tespiti | Geçmiş (ne var) |
| **Diagnostic** | Sebep-sonuç-etkileşim; korelasyon ve yorumla içgörü | Geçmiş (neden oldu) |
| **Predictive** | Geçmiş + mevcut veriyle gelecek olasılık modeli; trend/sapma | Gelecek (ne olabilir) |
| **Prescriptive** | Hangi aksiyonun alınacağını daraltır; sorunu önle/tetikle | Gelecek (ne yapmalı) |

Eşleştirme: Information Gathering + Vulnerability Research = **descriptive**. Bulunan zafiyeti teşhis = **diagnostic + predictive**.

### Gördüğün ≠ Sahip Olduğun

Modülün en öğretici örneği — açık **TCP port 2121**:
```
Ne görüyorum?  → Açık TCP 2121 (connection-oriented)
Standart mı?   → Hayır (well-known portlar 0-1023)
Tanıdık mı?    → Evet — "21" → FTP. Admin'ler sık sık "hatırlaması kolay" alternatif kullanır
Doğrula        → Netcat / FTP client ile bağlan
Gözlem         → Bağlantı ~15 sn sürdü → --min-rtt-timeout 15-20 ile tekrar tarayıp teyit et
```
**Ders:** Bir tarama sonucu ham veridir; çıkarım yapıp **doğrula veya çürüt**. Kesin sorular sor, bildiğin ile bilmediğini ayır.

### Vulnerability Research — Kaynaklar
CVEdetails, Exploit-DB, Vulners, Packet Storm Security, NIST. Bir servis versiyonundan CVE bulduysan, zafiyet büyük olasılıkla hâlâ var. PoC kodu çoğu zaman **kendi vakana uyarlanmalı** (admin'in manuel konfigürasyonları).

### Evasion gerekiyorsa
Hedef sistemi **lokalde birebir replika** et (aynı versiyonlar), zafiyeti orada ara — canlı sistemi alarm tetikleyerek yoklamadan.

### The Return — İki Aşama İç İçe
Zafiyet bulamazsan Information Gathering'e **geri dön**, daha derin bilgi topla. Bu iki aşama sürekli örtüşür, aralarında gidip gelinir.

> **Kritik felsefe:** A real Penetration Test is **not a CTF**. CTF'te hız > kalite (flag kap, çık). Gerçek testte **kalite ve derinlik** önceliklidir — çünkü basit bir vektörle hacklenen müşteri, o vektörü bizim bulmamız gerekirdi.

---

## 10. Exploitation

Bulunan zafiyetleri hedefe uyarlayıp çalıştırma — initial access, priv-esc vb. Reverse shell için PoC, hedef bize (ideal olarak encrypted) bağlanacak şekilde değiştirilir. Exploit hazırlığı **büyük ölçüde** bu aşamanın parçası.

> Aşamalar keskin ayrılmaz ama hangi fazda olduğunu bilmek şart — haftalarca süren, geniş scope'lu testte adımları kaybetmek çok kolay.

### Attack Prioritization — 3 Faktör

```
┌─ Probability of Success  → başarı olasılığı (CVSS / NVD calculator yardımcı olur)
├─ Complexity              → zaman + emek + araştırma (tecrübe belirleyici)
└─ Probability of Damage   → hedefe zarar riski (DoS'tan kaçın!)
```

### Puanlama Örneği

| Faktör | Puan | RFI | Buffer Overflow |
|---|---|---|---|
| Probability of Success | 10 | 10 | 8 |
| Complexity — Easy | 5 | 4 | 0 |
| Complexity — Medium | 3 | 0 | 3 |
| Complexity — Hard | 1 | 0 | 0 |
| Probability of Damage | −5 | 0 | −5 |
| **Toplam** | max 15 | **14** | **6** |

→ **RFI** tercih edilir: hazırlaması/çalıştırması kolay, dikkatli yapılırsa zarar yok.

### Hazırlık ve Altın Kural
- Kaliteli çalışan PoC yoksa → hedefi **lokal VM'de** yeniden kur (aynı versiyonlar), neyin değişeceğini bul, çalıştığından ve ciddi zarar vermediğinden emin ol.
- **DoS genelde yapılmaz** (müşteri istemedikçe). Canlı servise zarar verebilecek exploit her zaman kaçınılır.

> **When in doubt, communicate.** Riskli bir attack öncesi müşteriye/manager'a sor — bilgilendirilmiş karar versinler. "Yapma" derlerse rapora "aktif doğrulanmadı ama muhtemel issue" diye not düş. Ekstra iletişim, çökmüş sistemi ayağa kaldırmaya çalışmaktan iyidir.

Başarılı exploit + **net notlar ve activity log** → post-exploitation & lateral movement.

---

## 11. Post-Exploitation

Hedefteyiz; artık alarmdan kaçmak çok daha zor. Amaç: lokal perspektiften hassas/güvenlik-ilişkili ve business bilgisi toplamak — çoğunlukla standart user'dan yüksek yetki gerekir.

**Bileşenler:** Evasive Testing · Information Gathering · Pillaging · Vulnerability Assessment · Privilege Escalation · Persistence · Data Exfiltration

### Evasive Testing (3 kategori)

| Kategori | Ne zaman |
|---|---|
| **Evasive** | Tam gizlilik hedefi; blind spot'ları bul |
| **Hybrid-Evasive** | Önceden tanımlı belirli bileşen/departman/sunucu test edilir |
| **Non-Evasive** | Intrusive, maksimum derinlik; güvenlik önlemleri sınırlayabilir |

> Fark edilmek her zaman kötü değil — saldırganı emüle ediyoruz, bazı aksiyonların yakalanması müşterinin savunmasının çalıştığını gösterir. `net user`, `whoami` gibi komutlar EDR tarafından sık izlenir → dikkatsizce çalıştırmak ele verir.

### Alt Aşamalar

**Information Gathering (tekrar):** Artık yeni bir ortamdayız (iç perspektif). Dış perspektiften toplanan bilgi yetmez; lokal network, printer, DB server, virtualization servisleri yeniden enumerate edilir.

**Pillaging:** Host'un ağdaki rolünü incele —
```
Interfaces · Routing · DNS · ARP · Services · VPN · IP Subnets · Shares · Network Traffic
```
Buradan: alternatif subdomain'ler, çoklu NIC, iletişim kurulan diğer host'lar, admin bağlantıları, reusable credential / SSH key çalma, persistence imkânı. Password policy zayıfsa (8 karakter, özel karakter yok) → diğer kullanıcı parolalarını tahmin olasılığı yüksek. Sensitive data avı: share, script, config, vault, doküman, mail.

**Persistence:** Erişimi sürdür — bağlantı koparsa geri gel. Çoğu zaman Information Gathering'den **önceki ilk adım**. Örn. buffer overflow ile servis çökebilecekse, servise tekrar tekrar saldırmamak için hemen persistence kur.

**Vulnerability Assessment (içeriden):** Sistem bilgisiyle zafiyet analizini tekrarla, önceliklendir. Hedef: privilege escalation.

**Privilege Escalation:** Kritik an — çok kapı açar. Hedef: root (Linux) / Domain Admin / Local Admin / SYSTEM (Windows). **Lokal olmak zorunda değil** — information gathering'de bulunan yüksek yetkili başka kullanıcının credential'ıyla login olmak da priv-esc sayılır.

**Data Exfiltration:** Veriyi hedeften bize aktarabilir miyiz? DLP, EDR, network monitoring, disk encryption karşı önlem. **Önce müşteri/manager'a sor.** Çoğu zaman **sahte veri** (fake kredi kartı/SSN) üretip exfiltrate etmek yeterli — pattern tabanlı korumayı test eder, gerçek sensitive veriyi test makinemize almayız.

| Veri tipi | Regülasyon |
|---|---|
| Kredi kartı | PCI |
| Sağlık (PHI) | HIPAA |
| Banka (tüketici) | GLBA |
| Devlet bilgisi | FISMA |

Framework'ler: NIST, CIS Controls, ISO, PCI-DSS, GDPR, COBIT, FedRAMP, ITAR, AICPA, NERC CIP. Bizim için veri tipi değil, **etrafındaki kontroller** önemli.

> Kritik adımlarda **screen recording + screenshot**. Terminal-only ise hostname, IP, username, dosya yolunu ekranda göster → verinin kaynağını ve çıkarılabildiğini kanıtlarsın.

---

## 12. Lateral Movement

İç ağda yatay ilerleme — bir saldırganın tüm ağda neler yapabileceğini test etmek. En yaygın kâbus örneği: **ransomware** (tek host'tan tüm ağa yayılıp her şeyi encrypt eder).

Bu aşamaya hem Exploitation'dan hem Post-Exploitation'dan girilebilir. İçinde tekrar koşturulan fazlar: **Pivoting · Evasive Testing · Information Gathering · Vulnerability Assessment · (Privilege) Exploitation · Post-Exploitation**.

### Pivoting / Tunneling
Ele geçirilmiş host'u **proxy** gibi kullanıp tüm taramaları kendi makinemizden yapmak. Böylece **non-routable** (dışarıdan erişilemez) ağlara ulaşılır.

```
[Saldırgan VM] ──► [Compromised Host] ──► [İç ağ: non-routable hosts]
                    (proxy/pivot)
```
Basit analoji: internetten erişilemeyen ev yazıcısına, ele geçirilmiş bir ev host'u üzerinden print job göndermek.

### Diğer Fazlar (yatay bağlamda)
- **Evasive Testing:** network (micro)segmentation, threat monitoring, IPS/IDS, EDR nasıl çalışıyor anla → adapte ol.
- **Information Gathering:** İçeriden, hangi sistemlere erişilebiliyor haritala; sonra tek tek enumerate et.
- **Vulnerability Assessment:** İç ağda dış ağdan **çok daha fazla hata** olur — grup üyelikleri, paylaşım hakları kritik. Developer grubundaki bir hesabı ele geçirmek → tüm dev kaynaklarına erişim.
- **(Privilege) Exploitation:** Password/hash crack, credential reuse. Bazen hash crack bile gerekmez — **Responder** ile NTLMv2 hash intercept → admin hash'i yakalanırsa **pass-the-hash** ile birden çok host'a admin login.
- **Post-Exploitation (tekrar):** Her yeni host için baştan — sistem bilgisi, user verisi, business bilgisi; sensitive data kurallarını yine gözet.

---

## 13. Proof-of-Concept

**Tanım:** Projenin prensipte uygulanabilir olduğunun kanıtı. Bizim için: zafiyetin **var olduğunu** developer/admin doğrulayabilsin, reproduce edebilsin, etkiyi görsün, remediation'ını test edebilsin diye kanıtlamak. Klasik örnek: hedefte `calc.exe` çalıştırmak.

### PoC Biçimleri
- **Dokümantasyon** da bir PoC olabilir.
- Daha pratik: zafiyeti **otomatik exploit eden script/kod** — admin adım adım görür.

### Script'in Tuzağı
Admin'ler script'i alıp onunla "savaşır" — sistemi **sadece o script çalışmayacak** şekilde değiştirirler. Oysa script yalnızca **bir** exploit yolu; bilgi başka yolla da elde edilebilir. Bu nokta açıkça belirtilmeli.

### Attack Chain Mantığı
Domain compromise'de attack chain walkthrough koymak değerli: birden çok zafiyetin nasıl zincirlendiğini, **bir halkayı kırmanın zinciri bozduğunu ama diğer zafiyetlerin kaldığını** gösterir.

> **Örnek:** Kullanıcı `Password123` kullanıyorsa, asıl zafiyet **parola değil, password policy**. Domain Admin'in parolasını değiştirsen o hesap güçlenir ama zayıf parola sorunu kurumda endemik kalır. Yüksek standartlı policy olsaydı o parola hiç kullanılamazdı.

---

## 14. Post-Engagement

Testten sonraki (çoğu contractually binding) işler.

### Cleanup
Yüklenen tool/script'leri sil, (minor) config değişikliklerini geri al. Detaylı notlar temizliği kolaylaştırır. Erişilemeyen bir sistemde artifact kaldıysa → müşteriyi uyar, rapor appendix'ine yaz. Geri alınan değişiklikler (örn. eklenen local admin) yine de appendix'e yazılır — müşteri alarm alırsa "bu bizim sanctioned testimizdi" diye teyit edebilsin.

### Documentation & Reporting — Rapor İçeriği
```
☐ Attack chain (tam compromise / external→internal)
☐ Güçlü Executive Summary (teknik-olmayan kitle anlasın)
☐ Detaylı findings: risk rating + impact + remediation + external references
☐ Steps to reproduce (remediation ekibi test edebilsin)
☐ Near / medium / long-term öneriler (ortama özel)
☐ Appendices: scope, OSINT, password cracking, port/service, compromised hosts/accounts,
  transferred files, account creation/system mods, AD security analysis, scan data
```
İlk teslim **DRAFT** damgalı — müşteri yorumlar, açıklama/değişiklik ister.

### Report Review Meeting
Draft sonrası walkthrough. Rapor kelime kelime okunmaz; her finding kısaca + kendi perspektifimizle anlatılır. Müşteri soru sorar, düzeltme ister. Genelde low-risk'leri atlar, belirli finding'lere odaklanır.

### Deliverable Acceptance
Müşteri feedback'i sonrası **FINAL** damgalı yeni versiyon. Bazı audit firmaları DRAFT damgalı raporu kabul etmez — tutarlılık için tüm müşterilerde aynı yaklaşım.

### Post-Remediation Testing (Retest)
Çoğu engagement'ın parçası. Müşterinin remediation kanıtını incele, ortama tekrar eriş, her issue'yu test et. Before/after gösteren rapor:

| # | Severity | Finding | Status |
|---|---|---|---|
| 1 | High | SQL Injection | Remediated |
| 4 | High | Inadequate Web/Egress Filtering | **Not Remediated** |
| 5 | Medium | SMB Signing Not Enabled | **Not Remediated** |

Mümkünse: scan output veya orijinal exploit'in artık **başarısız** olduğunun kanıtı.

### Pentester'ın Remediation Rolü
Pentest bir **audit** — impartial 3rd party kalmalıyız. **Remediation yapmayız** (kod düzeltmek, patch, AD config değiştirmek yok). Genel öneri veririz ("sanitize user input") ama **kesin çözüm yazmayız** (yeniden yazılmış kod vermeyiz). Conflict of interest'ı önler.

### Data Retention
Test sonrası elde: scan, log, credential, screenshot. Retention/destruction kuralları ülke/firma ve SoW+RoE'ye göre. PCI DSS: delil bir süre saklanması **tavsiye** (zorunlu değil), talep üzerine erişilebilir olmalı. Saklanan veri → firma kontrolünde güvenli konum, **encrypted at rest**. Tester sistemlerinden tüm veri **silinir**. Retest için müşteriye özel yeni VM.

### Close Out
Final rapor + remediation desteği + retest sonrası proje kapanır. Sistemler wipe/destroy, artifact'lar encrypted saklanır. Son adımlar: invoice + ödeme, ve client satisfaction survey.

> Müşteri genelde fancy exploit chain'i değil, **etkileşimleri, iletişimi ve nasıl değer gördüğünü** hatırlar. Soft skill'leri sürekli geliştir.

---

# Bölüm IV — Kapanış

## 15. Pratik Metodolojisi

Teori tek başına yetmez — hands-on + güçlü yazılı/sözlü iletişim şart. Önerilen başlangıç dozu:

```
2x Modül  +  3x Retired Machine  +  5x Active Machine  +  1x Pro Lab / Endgame
```

### Modül Çalışma Blueprint'i
```
1. Modülü oku
2. Exercise'leri çöz
3. Modülü tamamla
4. Exercise'leri SIFIRDAN tekrar başlat
5. Tekrar çözerken NOT al
6. Notlardan teknik dokümantasyon
7. Notlardan teknik-olmayan dokümantasyon
```

### Retired Machine Akışı (write-up erişimi avantajı)
```
1. User flag (kendi başına)
2. Root flag (kendi başına)
3. Teknik dokümantasyon
4. Teknik-olmayan dokümantasyon
5. Resmi/community write-up ile karşılaştır
6. Kaçırdıklarını listele
7. Ippsec walkthrough'ını izle, karşılaştır
8. Eksikleri ekleyerek notları genişlet
```
> Retired'da VIP → resmi HTB write-up'ları + defensive considerations.

### Active Machine Akışı (blackbox — write-up yok)
```
1. User + root flag
2. Teknik dokümantasyon
3. Teknik-olmayan dokümantasyon
4. Teknik VE teknik-olmayan kişiye proofread ettir
```
> Önerilen zorluk dağılımı: 2 easy + 2 medium + 1 hard.

### Pro Lab / Endgame
Çok-host'lu enterprise ortam simülasyonu. Artık tek host değil, host'ların **birbiriyle etkileşimi** önemli (örn. AD'de Responder ile hash capture). Tek makine değil, **tüm attack chain** yazılır (foothold → network compromise).

> **Documentation öğretir:** Dokümantasyon yazmak, konuyu zihne kazır ve hem teknik hem teknik-olmayan kitleye anlatmayı kolaylaştırır. Reading ≠ hands-on practice.

---

## 16. Tek Bakışta Özet

| # | Aşama | Özü | Kritik hatırlatma |
|---|---|---|---|
| 1 | Pre-Engagement | Contract + scope + RoE yazılı bağla | NDA önce; sadece yetkili imzalar; avukat gözden geçirir |
| 2 | Information Gathering | 4 kategoriyle her bilgiyi topla | Temel taş; sürekli geri dönülür; pillaging buranın parçası |
| 3 | Vulnerability Assessment | Analiz et, attack vector çıkar | Gördüğün ≠ sahip olduğun; doğrula; **pentest ≠ CTF** |
| 4 | Exploitation | Uyarla, initial access al | 3 faktörle prioritize; DoS'tan kaçın; şüphede iletişim |
| 5 | Post-Exploitation | Priv-esc, pillage, persistence | Priv-esc lokal olmak zorunda değil; exfil'de fake data |
| 6 | Lateral Movement | Pivot + yatay ilerle | Pivoting ile non-routable ağa ulaş; pass-the-hash |
| 7 | Proof-of-Concept | Zafiyeti kanıtla, zincirle | Asıl zafiyet policy'dir; script tek yoldur |
| 8 | Post-Engagement | Raporla, temizle, arşivle | Remediation YAPMA; DRAFT→FINAL; encrypted retention |

---

## Kaynaklar

Notlar aşağıdaki kaynaklardan derlendi:

- **HTB Academy** — Penetration Testing Process (CPTS path, Module 01)
- **OSSTMM / PTES / OWASP** — pentest metodoloji standartları
- **CVSS / NIST NVD** — zafiyet puanlama ve prioritization
- Yasal çerçeveler: **GDPR, CFAA, DMCA, HIPAA, COPPA, Computer Misuse Act**
- Veri regülasyonları: **PCI-DSS, GLBA, FISMA**
- Vulnerability kaynakları: CVEdetails, Exploit-DB, Vulners, Packet Storm Security, NIST

Notlar kişisel çalışma amacıyla tutuldu ve halka açık paylaşıldı.
Hata veya eksik gördüğünüz yerler için issue açabilirsiniz.

[← Tüm modüller](../README.md)
