---
title: Kurumsal Strateji, Olgunluk Modeli ve ROI
tags:
  - strategy
  - roi
  - zerotrust
  - maturity-model
  - governance
status: approved
version: "3.0"
updated: 2026-10-19
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Kurumsal Strateji, Olgunluk Modeli ve Teknik Katma Değer Analizi

Bu belge; kurumsal SAP ekosisteminde kimlik ve erişim yaşam döngüsü (JML) otomasyonunun olgunluk seviyelerini, NIST SP 800-207 Sıfır Güven (Zero Trust) standardı uyumunu, doğuştan gelen (birthright) erişim sınırlarını ve operasyonel verimlilik kriterlerini teknik ve akademik bir referans çerçevesinde belgeler.

> [!NOTE]
> **Giriş ve Kapsam:**  
> Bu araştırma çalışması, SAP canlıya geçiş sürecinde kurumumuzun mevcut kurumsal kimlik yatırımlarını (SF $\rightarrow$ AD) koruyarak SAP ekosistemine bağlamayı amaçlamaktadır. Sistemin tıkanmaması için ilk etapta tüm modüllerin en ince yetki ayrıntılarını çözmek yerine; **Doğuştan Gelen (Birthright) yetkiler ve anında hesap kapatma (Leaver) omurgasıyla başlanması** önerilmektedir. Sistemde manuel verilmiş mevcut yetkileri koruyan parametreler (`authoritative = false`, `delete.threshold = 10`) devreye alınarak hem iş sürekliliği riske atılmayacak, hem de 13 Kasım 2026 SAP IAS regülasyonuna tam uyum sağlanacaktır.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Keşif Kılavuzu: [[02 - Teknik Keşif Anketi (Workshop Guide)]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* Güvenlik ve Koruyucu Parametreler: [[05 - Güvenlik, Denetim ve Break-Glass]]

---

## 1. IAM Kurumsal Olgunluk Modeli (IAM Maturity Model)

Kurumların kimlik ve erişim yönetimi süreçleri endüstri standardı 4 olgunluk seviyesinde sınıflandırılır:

```
[ Seviye 1: Kaotik / Manuel ] ──► [ Seviye 2: Parçalı / IT Odaklı ] ──► [ Seviye 3: Kural Tabanlı JML ] ──► [ Seviye 4: Sıfır Güven & AI ]
- IT'ye açılan biletler          - AD scriptleri var                 - SuccessFactors tetikler            - Risk tabanlı adaptif erişim
- Excel'de tutulan yetkiler      - SAP sistemleri bağımsız           - Sıfır manuel bilet (Zero-Touch)     - Otomatik SoD denetimi (IAG)
- Haftalar süren onaylar         - Yetki birikimi ve kayması yüksek  - Kural tabanlı otomatik provizyon   - Anomali ve bağlam analizi
```

### Seviye Analizi ve Teknik Çerçeve:
* **Seviye 1 (Kaotik / Manuel - Reaktif):** İşe girişlerde İK, IT'ye e-posta/bilet açar. SAP Basis uzmanı `SU01` ekranında manuel kullanıcı ve rol tanımlar. İşten ayrılanların yetkileri haftalarca kapatılamaz; yetim hesap (orphan account) oranı %10'u aşar.
* **Seviye 2 (Parçalı / Silo Yapı):** Active Directory tarafında bazı otomatikleştirilmiş scriptler bulunur; ancak SAP ekosistemi (S/4HANA, BTP, SAC) bu otomasyonun dışındadır. Senkronizasyon manuel biletlerle sağlanır.
* **Seviye 3 (Kural Tabanlı Otomasyon / JML Standardı):** İK ana veri kaynağındaki (SAP SuccessFactors) sözleşme veya pozisyon değişikliği tek güven kaynağıdır (Single Source of Truth). Entra ID, SAP IAS, BTP ve S/4HANA üzerinde roller **Zero-Touch** olarak otomatik açılır, güncellenir ve dondurulur.
* **Seviye 4 (Sıfır Güven & AI Destekli - İleri Düzey):** Kimlikler yalnızca statik unvana göre değil; kullanıcının anlık bağlamına (cihaz durumu, IP, anomali skoru) ve SAP Cloud Identity Access Governance (IAG) ile gerçek zamanlı Görevler Ayrılığı (SoD) kontrollerine göre dinamik olarak yönetilir.

---

## 2. NIST SP 800-207 "Sıfır Güven" (Zero Trust) Hizalanması

Bu mimari model, ABD Ulusal Standartlar ve Teknoloji Enstitüsü'nün (NIST) **SP 800-207 Zero Trust Architecture** [^1] standardında tanımlanan 3 çekirdek ilkeyi eksiksiz uygular:

### A. Ağa Değil, Kimliğe Güven (Explicit Verification)
* **Kural:** Şirket iç ağında (LAN) veya kurumsal VPN'de olmak kullanıcıya varsayılan olarak SAP sistemlerine erişim hakkı **tanımaz**.
* **Uygulama:** Her istekte Microsoft Entra ID $\rightarrow$ SAP Cloud Identity Services (IAS) $\rightarrow$ SAP BTP XSUAA zincirinde kimlik, token geçerliliği ve güvenlik grubu üyelikleri açıkça doğrulanır.

### B. En Az Yetki İlkesi (Principle of Least Privilege)
* **Kural:** Kullanıcıya hiçbir zaman "geniş/jenerik" veya "ne olur ne olmaz" yaklaşımıyla rol atanmaz.
* **Uygulama:** Yalnızca SuccessFactors üzerindeki doğrulanmış görev tanımına (`JobCode`) karşılık gelen atomik SAP iş rolleri ve türetilmiş organizasyonel kısıtlar verilir (Bkz: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü#3-rol-ve-yetki-mimarisi-yetki-tasarımı-modül-içi-vs-dağıtımı-ips-ve-veri-kısıtı]]).

### C. Kısa Ömürlü İtimat (Ephemeral Trust & Principal Propagation)
* **Kural:** Sistemler arasında statik teknik kullanıcı şifreleri (Hardcoded Service Passwords) paylaşılmaz.
* **Uygulama:** SAP BTP'den arka uç S/4HANA sistemine erişirken SAP Cloud Connector (SCC) tarafından dinamik olarak üretilen **2 dakikalık geçici X.509 istemci sertifikaları** kullanılır [^2]. Bu sayede çalınabilecek statik bir şifre bulunmaz ve denetim izinde işlemi yapan gerçek kullanıcının kimliği (Principal) görünür.

---

## 3. Erişim Sınırları: Doğuştan Gelen (Birthright) vs. Talep Tabanlı Yetki Ayrımı

Kurumsal mimaride yetki enflasyonunu önlemek adına yetkiler kesin sınırlarla ikiye ayrılmıştır:

```
                                  ┌─────────────────────────────────────────┐
                                  │       ÇALIŞANIN YETKİ EVRENİ             │
                                  └─────────────────────────────────────────┘
                                                       │
                 ┌─────────────────────────────────────┴─────────────────────────────────────┐
                 ▼                                                                           ▼
┌──────────────────────────────────────────────┐                            ┌──────────────────────────────────────────────┐
│        1. DOĞUŞTAN GELEN YETKİLER             │                            │         2. TALEP TABANLI YETKİLER            │
│             (Birthright Access)              │                            │            (Request-Based Access)            │
├──────────────────────────────────────────────┤                            ├──────────────────────────────────────────────┤
│ - SuccessFactors pozisyonu ile otomatik gelir.│                            │ - Pozisyonla doğrudan gelmez; geçicidir.     │
│ - Örn: ESS (İzin alma), Bordro görüntüleme,   │                            │ - Örn: Proje bazlı sistem açılışı, acil      │
│   Departmanına özel standart muhasebe ekranı.│                            │   müdahale yetkisi, bütçe aşım onayı.        │
│ - [Merkezi IAM Otomasyon Kapsamı]            │                            │ - Yönetici onayı + SoD kontrolü gerektirir.   │
└──────────────────────────────────────────────┘                            └──────────────────────────────────────────────┘
```

> [!NOTE]
> **Mimari Kapsam ve Yetki Sınırları:**  
> Kimlik yaşam döngüsü otomasyonu, çalışanların sözleşmelerine ve departmanlarına bağlı **Doğuştan Gelen (Birthright)** yetkileri hedefler. Finansal limit artırımı, acil müdahale (Firefighter) veya proje bazlı geçici yetkiler SAP IAG / GRC onay akışlarına tabidir.

---

## 4. Operasyonel Metrikler ve Teknik Katma Değer Kriterleri

Kimlik yaşam döngüsü otomasyonunun kurumsal operasyonlara ve sistem güvenliğine sağladığı ölçülebilir kazanımlar:

| Metrik / KPI | Manuel Süreç (Mevcut Durum) | Otomasyon Sonrası (Hedeflenen) | Sağlanan Teknik & Operasyonel Değer |
|:---|:---|:---|:---|
| **Erişim Hazırlık Süresi (Time-to-Productivity)** | **3 ila 7 iş günü** (Bilet onayları, SAP Basis bekleme süreleri). | **< 1 Saat (Day-1 Ready)** | Yeni çalışan ilk mesai gününde tüm SAP ekranlarına erişerek anında katma değer üretmeye başlar. |
| **IT Destek Yükü (Ticket Volume)** | Ayda yüzlerce "SAP yetki talebi", "şifre sıfırlama", "hesap açılışı" bileti. | **%80 - %90 Azalma** | IT ve SAP Basis ekipleri operasyonel bilet çözmek yerine stratejik projelere odaklanır. |
| **Yetim Hesap (Orphan Account) Oranı** | İşten ayrılanların ortalama %5-10'unun SAP hesapları haftalarca açık kalır. | **%0 (Sıfır Tolerans)** | Sözleşme feshedildiği saniye tüm oturumlar kilitlenir; eski çalışan veri sızıntısı riski son bulur. |
| **Denetim Hazırlık Süresi (Audit Readiness)** | Yılda 2 kez yapılan denetimlerde ekran görüntüleri ve Excel logları toplamak haftalar sürer. | **Dakikalar İçinde Tek Tıkla** | Denetçilere IPS senkronizasyon logları ve IAS Audit Trail doğrudan resmi delil olarak sunulur. |

---

## 5. Kademeli İlerleme Modeli (3 Aşama)

Şirketin tamamını aynı anda mükemmel bir otomasyona geçirmek imkansızdır. Kademeli ve güvenli ilerleme modeli:

| Aşama | Odak Noktası | Ne Yapılır? | Risk Düzeyi |
|:---|:---|:---|:---|
| **Adım 1: Sadece Giriş & Çıkış (Joiner & Leaver)** | Temel Güvenlik ve İmaj | Yeni girenlerin standart hesapları açılsın; ayrılan personelin tüm SAP hesapları anında kilitlensin. Departman içi karmaşık rol değişikliklerine (Mover) henüz girilmez. | **Çok Düşük** (En hızlı kazanım). |
| **Adım 2: Departman Rollerinin Otomasyonu** | Operasyonel Verimlilik | İK ve IT tarafında unvanlar oturdukça, departman bazlı standart SAP paketleri AD gruplarına bağlanır. 15 günlük çift yetki toleransı (Grace Period) işletilir. | **Orta** |
| **Adım 3: İnce Taneli Veri Kısıtları (ABAC)** | Derin Güvenlik | Roller oturduktan sonra, şirket kodu veya masraf merkezi gibi veri kısıtlamaları sisteme dinamik olarak entegre edilir. | **İleri Seviye** |

---

## 6. İleriye Dönük Tavsiyeler ve İnovasyon Fikirleri (Roadmap Notları)

Gelecek faz değerlendirmeleri için teknik yol haritası notları:

* **Dinamik AD Grupları:** IT birimi ileride Microsoft Entra ID'nin "Dynamic Membership" özelliğini kullanarak; çalışanın unvanı veya departmanı değiştiğinde AD grubuna ekleme işini bile kural bazlı yapabilir (IT personelinin elle gruba eklemesine gerek kalmaz).
* **Görevler Ayrılığı (SoD) ve SAP IAG:** Gelecekte riskli finansal roller otomasyona alınmak istendiğinde, araya bir onay ve çakışma kontrolü katmanı koymak için **SAP Cloud Identity Access Governance (IAG)** entegrasyonu değerlendirilebilir.
* **Periyodik Yetki Gözden Geçirme (Access Recertification):** Yılda bir kez birim yöneticilerine *"Ahmet hala bu SAP ekranlarını kullanıyor mu?"* onayının otomatik gönderildiği bir sertifikasyon akışı kurgulanabilir.

---

## Resmi Referanslar ve Kaynakça

[^1]: **NIST Special Publication 800-207:** Rose, S., Borchert, O., Mitchell, S., & Connelly, S. (2020). *Zero Trust Architecture*. National Institute of Standards and Technology. [https://doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207)
[^2]: **SAP Help Portal - Connectivity Service:** *Configuring Principal Propagation to an ABAP System for HTTPS with Cloud Connector*. [https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/configuring-principal-propagation-to-abap-system-for-https](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/configuring-principal-propagation-to-abap-system-for-https)
[^3]: **The Open Group:** *TOGAF Standard, 10th Edition - Identity and Access Management Architecture Pattern*. [https://www.opengroup.org/togaf](https://www.opengroup.org/togaf)
