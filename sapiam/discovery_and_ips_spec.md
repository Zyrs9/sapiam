# SAP IAM & JML Kurumsal Mimari, Entegrasyon ve Teknik Referans Kılavuzu (Monolith Blueprint)

**Doküman Versiyonu:** 3.1 (Teknik Entegrasyon, Standartlar ve En İyi Uygulamalar Konsolide Sürümü)  
**Tarih:** 2026-10-22  
**Kapsam:** SAP SuccessFactors $\rightarrow$ Microsoft Entra ID $\rightarrow$ SAP Cloud Identity Services (IAS/IPS) $\rightarrow$ SAP BTP & S/4HANA  
**Doküman Niteliği:** Kurumsal Kimlik & Erişim Yaşam Döngüsü (JML) Canlıya Geçişi için Teknik Referans ve Mimari Standartlar Kılavuzu  
**Standart Uyumu:** IETF RFC 7643 / 7644 (SCIM 2.0), NIST SP 800-207 (Zero Trust), SAP Identity Directory API, ISO/IEC 27001:2022  

---

## Giriş ve Kapsam

Bu çalışma; şirketimizin SAP'ye geçiş sürecinde kimlik doğrulama, kullanıcı yaşam döngüsü (JML) ve yetki dağıtım mekanizmalarının nasıl kurgulanabileceğini araştırmak amacıyla hazırlanmıştır. Mevcut durumda İK ve BT ekiplerimizce işletilen SuccessFactors $\rightarrow$ Active Directory (AD) yapısını koruyarak SAP sistemlerine güvenli ve kesintisiz biçimde bağlamak hedeflenmektedir. Bu dokümanda, canlıya geçiş sürecinde sistemlerin tıkanmaması ve sahadaki teknik kısıtların aşılabilmesi için derlenen bulgular, parametre önerileri ve mimari standartlar ekibimizin değerlendirmesine sunulmuştur.

---

## Canlıya Geçiş Öncesinde Tanımlanması Önerilen Koruyucu Parametreler

Sistemin aksamaması ve geçiş sırasında beklenmeyen yetki kayıplarının yaşanmaması için hedef sistemlerde önceden tanımlanması faydalı görülen 4 temel koruyucu parametre bulunmaktadır:

1. **Mevcut Yetkilerin Silinmesini Engelleyen Yapılandırma (`s4hana.authoritative.role.management = false`):**
   * *Teknik Gerekçe:* Canlıya geçiş aşamasında otomasyon doğrudan ezici (authoritative) modda açılırsa; sistem, Active Directory gruplarında yer almayan ancak personelin S/4HANA'da işini yürütebilmesi için elle atanmış mevcut yetkilerini silebilir.
   * *Öneri:* İlk aşamada sistemin sadece AD'den gelen yeni yetkileri eklemesi, içeride tanımlı mevcut rollere ise dokunmaması (`authoritative = false`) iş sürekliliği açısından daha güvenli bir yol olacaktır.

2. **Toplu Silinme veya Kilitlenmeleri Durduran Eşik Değeri (`ips.delete.threshold = 10`):**
   * *Teknik Gerekçe:* Dizin tarafında yapılacak bir betik hatasında veya bir organizasyonel birimin (OU) yanlışlıkla taşınması durumunda, senkronizasyon motoru personeli toplu olarak işten ayrılmış algılayabilir.
   * *Öneri:* Tek bir senkronizasyon döngüsünde kullanıcıların %10'undan fazlasını kilitlemeye veya silmeye yönelik bir durum oluştuğunda IPS işleminin otomatik olarak durması (`ips.delete.threshold = 10`) sistemsel bir güvenlik bariyeri sağlar.

3. **Acil Müdahale ve Özel Yetki Filtresi (`Z_FF_*` / `Z_EXC_*`):**
   * *Teknik Gerekçe:* Canlıya geçişin ilk günlerinde acil durumlar için SAP Basis ve yetki ekiplerince geçici teknik roller açılması gerekebilir. Bu rollerin otomasyon tarafından kaldırılmaması gerekir.
   * *Öneri:* Teknik adı `Z_FF_` veya `Z_EXC_` ile başlayan rollerin IPS hedef filtrelerinde istisna tutulması (`ips.target.options.role.exclude.filter = (Z_FF_.*|Z_EXC_.*)`) sahada operasyonel esneklik sağlayacaktır.

4. **Değişmez Ortak Anahtar Alanı (`employeeID`):**
   * *Teknik Gerekçe:* Eşleştirmeyi e-posta veya ad-soyad üzerinden yapmak; soyadı değişikliklerinde veya e-posta formatı güncellemelerinde kullanıcı eşleşmelerinin kopmasına yol açabilir.
   * *Öneri:* Sistemler arası kimlik bağının SuccessFactors'teki Personel Numarası (`personIdExternal`) ve AD'deki `employeeID` alanı üzerinden tekil anahtar olarak kurgulanması sağlıklı olacaktır.

---

## Önceden Netleştirilmesi Faydalı Olan Temel Konular

Süreçleri planlarken paydaşlar ve modül danışmanlarıyla şu 4 konunun çerçevesini baştan netleştirmemiz faydalı olacaktır:

```
                  ┌─────────────────────────────────────────────────────────┐
                  │             OTOMASYON KAPSAMI VE YAKLAŞIM               │
                  └─────────────────────────────────────────────────────────┘
                                               │
               ┌───────────────────────────────┴───────────────────────────────┐
               ▼                                                               ▼
┌─────────────────────────────────────────────┐ ┌─────────────────────────────────────────────┐
│          1. OTOMATİK AKACAK ALAN            │ │           2. MANUEL KALACAK ALAN            │
│            (Birthright Yetkileri)           │ │           (Riskli / Özel Yetkiler)          │
├─────────────────────────────────────────────┤ ├─────────────────────────────────────────────┤
│ • İzin alma, bordro görüntüleme (ESS)       │ │ • Finansal ödeme onayları (F110 vb.)        │
│ • Departmanın standart operasyonel ekranları │ │ • Satıcı ana verisi açma / değiştirme      │
│ • BTP üzerindeki kurumsal portallar         │ │ • Özel proje bazlı istisnai yetkiler        │
└─────────────────────────────────────────────┘ └─────────────────────────────────────────────┘
```

1. **Yetki Tasarımı vs. Yetki Dağıtımı:** Biz yetki tasarlamayacağız, S/4HANA tarafındaki yetkileri danışmanlar PFCG içinde hazırlayacak. Biz sadece bu rollerin teknik isimlerini alıp Active Directory gruplarına bağlayacağız. Böylece modül içine girip tek tek elle rol atama işi ortadan kalkacak.
2. **Otomasyon Sınırları (%80 vs %20):** Tüm yetkileri ilk günden otomatikleştirmek yerine; standart işe giriş ekranları (izin, bordro, departman ana ekranı) otomatik gelsin. Riskli finansal onaylar veya özel yetkiler için yine eskisi gibi onay formu işletilsin, böylece denetim riski oluşmasın.
3. **MFA Yaklaşımı:** Hem Microsoft'ta hem SAP'de iki kere kod sormasın; tüm giriş güvenliği Microsoft/AD üzerinde kalsın, SAP tarafı aradan çekilsin.
4. **Mevcut Kullanıcılar (Brownfield):** Canlıya geçmeden önce eski S/4HANA kullanıcılarının personel numarası alanlarını güncellememiz gerekiyor, yoksa sistem bu kullanıcıları tanıyamayıp hata verebilir.

---

## Kademeli İlerleme Modeli (3 Aşama)

Gözlemlediğimiz kadarıyla tüm yetkileri aynı anda devreye almak yerine süreci kademeli olarak ilerletmek riskleri azaltmaktadır:

| Aşama | Odak Alanı | Kapsam | Dikkat Edilmesi Gereken Nokta |
|:---|:---|:---|:---|
| **Aşama 1: Giriş ve Çıkış (Joiner & Leaver)** | Temel Güvenlik ve Hesap Açılışı | İşe giren personelin standart hesaplarının açılması ve işten ayrılan personelin hesaplarının aynı gün kilitlenmesi. | En hızlı devreye alınabilecek ve denetim güvenliğini sağlayacak temel adımdır. |
| **Aşama 2: Departman Düzeyinde Standart Roller** | Operasyonel Verimlilik | İK unvan ve departman yapıları netleştikçe, departmana özel standart paketlerin AD gruplarıyla bağlanması. Görev değişimlerinde 15 günlük geçiş toleransı (Grace Period) tanımlanabilir. | Modül danışmanlarıyla PFCG rol isimlerinin önceden netleştirilmesi gerekir. |
| **Aşama 3: İnce Taneli Veri Kısıtları (ABAC)** | Detaylı Yetkilendirme | Temel roller oturduktan sonra, şirket kodu veya masraf merkezi gibi veri düzeyi yetkilerinin sisteme dinamik olarak bağlanması. | Karmaşıklığı artırmamak için ilk fazda zorunlu tutulmaması önerilir. |

---

## İleriye Dönük Notlar ve Yol Haritası

İlerleyen dönemde değerlendirilebilecek teknik geliştirmeler:

* **Dinamik AD Grupları:** İlerleyen fazlarda Microsoft Entra ID'nin dinamik üyelik (Dynamic Membership) özelliği kullanılarak; personelin unvanı veya departmanı değiştiğinde AD gruplarına atama işlemi de kural bazlı olarak otomatikleştirilebilir.
* **Görevler Ayrılığı (SoD) ve SAP IAG:** Riskli finansal roller otomasyona dahil edilmek istendiğinde, sistemler arası yetki çakışmalarını denetlemek için SAP Cloud Identity Access Governance (IAG) entegrasyonu gündeme alınabilir.
* **Periyodik Yetki Gözden Geçirme:** Belirli aralıklarla yöneticilere personelin mevcut SAP ekran yetkilerini doğrulatan otomatik bir gözden geçirme (recertification) mekanizması kurgulanabilir.

---

## İÇİNDEKİLER

1. [Mimari Omurga: BTP Yetkilendirme Modeli ve SCIM 2.0 Standartları](#1-mimari-omurga-btp-yetkilendirme-modeli-ve-scim-20-standartları)
2. [Genişletilmiş Teknik Keşif ve Gereksinim Analiz Matrisi (27 Maddelik Soru Envanteri)](#2-genişletilmiş-teknik-keşif-ve-gereksinim-analiz-matrisi)
3. [İK Yaşam Döngüsündeki Uç Senaryolar (JML Edge Cases)](#3-ik-yaşam-döngüsündeki-uç-senaryolar-jml-edge-cases)
4. [Rol ve Yetki Mimarisi: İş Bölümü (Authoring vs. Provisioning) ve Veri Kısıtı](#4-rol-ve-yetki-mimarisi-iş-bölümü-authoring-vs-provisioning-ve-veri-kısıtı)
5. [Taktiksel Operasyon Katmanı: Senaryo Yapılabilirlik Matrisi, Paydaş Ön Koşulları ve Sınır Kılavuzu](#5-taktiksel-operasyon-katmanı-senaryo-yapılabilirlik-matrisi-paydaş-ön-koşulları-ve-sınır-kılavuzu)
6. [Doğrulanmış SAP IPS JSON Kod Spesifikasyonu (Tüm Sistemler)](#6-doğrulanmış-sap-ips-json-kod-spesifikasyonu)
7. [Uçtan Uca Teknik Kurulum, Entegrasyon ve Doğrulama Kılavuzu (5 Fazlı Rehber)](#7-uçtan-uca-teknik-kurulum-entegrasyon-ve-doğrulama-kılavuzu-5-fazlı-rehber)
8. [Kritik Güvenlik, Denetim ve Operasyonel IPS Parametreleri](#8-kritik-güvenlik-denetim-ve-operasyonel-ips-parametreleri)
9. [Kurumsal Kimlik Olgunluk Modeli ve Sıfır Güven (Zero Trust - NIST SP 800-207) Mimarisi](#9-kurumsal-kimlik-olgunluk-modeli-ve-sıfır-güven-zero-trust---nist-sp-800-207-mimarisi)
10. [İstisnai Erişim Yönetimi: Görevler Ayrılığı (SoD) ve Onay Mekanizmaları](#10-istisnai-erişim-yönetimi-görevler-ayrılığı-sod-ve-onay-mekanizmaları)
11. [Operasyonel Metrikler ve Teknik Katma Değer Analizi](#11-operasyonel-metrikler-ve-teknik-katma-değer-analizi)
12. [Olağanüstü Durum Müdahale (Break-Glass), Felaket Kurtarma ve KVKK/GDPR Uyum Çerçevesi](#12-olağanüstü-durum-müdahale-break-glass-felaket-kurtarma-ve-kvkkgdpr-uyum-çerçevesi)
13. [Kritik Regülasyon ve Uyumluluk Takvimi (SAP SuccessFactors Basic Auth Deprecation)](#13-kritik-regülasyon-ve-uyumluluk-takvimi)
14. [Kapsamlı Terimler ve Kısaltmalar Sözlüğü (Glossary & Acronyms)](#14-kapsamlı-terimler-ve-kısaltmalar-sözlüğü-glossary--acronyms)
15. [Konsolide Resmi Standartlar, RFC'ler ve Kaynakça Tablosu](#15-konsolide-resmi-standartlar-rfcler-ve-kaynakça-tablosu)

---

## 1. MİMARİ OMURGA: BTP YETKİLENDİRME MODELİ VE SCIM 2.0 STANDARTLARI

### 1.1. BTP Yetkilendirme Modeli (JIT Federation vs. IPS Provisioning)

Yaptığımız teknik incelemelere göre, kurumsal kimlik doğrulama (Authentication) ile arka plan yetkilendirme/provizyon (Authorization & Provisioning) süreçlerini birbirinden ayırmak sistem performansı açısından en sağlıklı yöntemdir:

* **BTP Hibrit Yetkilendirme Ayrımı:** Son kullanıcılar için **Yol A (Runtime OIDC Federation / Just-In-Time)** modelinin seçilmesi, BTP Subaccount veritabanında güncelliğini yitiren gölge kullanıcı (Shadow User) birikmesini engeller ve Entra ID'deki grup değişikliklerinin bir sonraki oturumda anında BTP'ye yansımasını sağlar. Sistem yöneticileri ve arka plan işleri (Batch Job / Workload) için **Yol B (IPS Direct Provisioning)** modelinin istisnai olarak konumlandırılması teknik gereksinimleri karşılamaktadır.

```
                               ┌────────────────────────────────────────────────────────┐
                               │           ENTRA ID / ACTIVE DIRECTORY                  │
                               │           Güvenlik Grupları (SG_SAP_*)                 │
                               └──────────────────────────┬─────────────────────────────┘
                                                          │
                                                          ▼
                                ┌────────────────────────────────────────────────────────┐
                                │       SAP CLOUD IDENTITY SERVICES - IAS (HUB)          │
                                └──────────┬──────────────────────────────────┬──────────┘
                                           │                                  │
                   [YOL A: Runtime OIDC Federation]                   [YOL B: Scheduled SCIM ETL]
                   (Token-Based / Just-In-Time)                       (Arka Plan / API Provisioning)
                                           │                                  │
                                           ▼                                  ▼
                     ┌───────────────────────────┐                      ┌───────────────────────────┐
                     │     SAP BTP (XSUAA/AMS)   │                      │        SAP S/4HANA        │
                     │   Role Collection Mapping │                      │   PFCG / Business Roles   │
                     │ (Uygulama Kullanıcıları)  │                      │    (Arka Plan Provizyon)  │
                     └───────────────────────────┘                      └───────────────────────────┘
```

#### A. Yol A (Önerilen - Uygulama Son Kullanıcıları): Just-In-Time (JIT) Token Federation
* **Çalışma Prensibi:** BTP üzerindeki SAP Fiori Launchpad, SAP Build Workzone, Cloud Application Programming Model (CAP) veya SaaS uygulamalarına erişen son kullanıcılar **IPS ile BTP veritabanına kopyalanmaz**. 
* **Mekanizma:**
  1. Kullanıcı SAP IAS üzerinden kurumsal kimliği (Entra ID) ile oturum açar.
  2. IAS, kullanıcının üye olduğu `SG_SAP_*` gruplarını OIDC ID Token / SAML Assertion içine `Groups` claim'i olarak ekler.
  3. BTP Subaccount seviyesinde tanımlanan **Role Collection Mapping** kuralları (XSUAA/AMS), bu token claim'ini okur ve kullanıcıya oturum anında rol koleksiyonunu (Role Collection) bağlar.
* **Sağladığı Kolaylık:** BTP kullanıcı veritabanında gereksiz kayıt birikmez; AD'de grup güncellendiği anda bir sonraki oturumda BTP yetkileri kendiliğinden güncellenmiş olur.

#### B. Yol B (İstisnai - Yönetimsel ve Servis Kullanıcıları): IPS Direct Provisioning
* **Kullanım Alanı:** Kullanıcının henüz sisteme hiç oturum açmadan (Day-0) BTP Subaccount Administrator, Cloud Connector Administrator olması gerektiğinde veya arka planda çalışacak Batch Job / Workload kimlikleri için IPS'in `SAP BTP Subaccount (Cloud Foundry)` hedef bağlayıcısı üzerinden XSUAA REST API ile ön-provizyon yapılır.

---

### 1.2. SCIM 2.0 Şema ve Dönüşüm Mantığı Kriterleri

Teknik tarafta SCIM 2.0 entegrasyonu kurgulanırken şu kurallara dikkat edilmesi sistemin kararlılığı için önem taşımaktadır:

* **Artımlı Senkronizasyon (Delta Sync):** Kaynak ve hedef dönüşümlerde `entityIdExternal`, `groupIdExternal` ve `entityIdTargetSystem` değişkenlerinin tanımlanmış olması, her döngüde tüm dizinin taranmasını engelleyerek yalnızca değişen kayıtların işlenmesini sağlar.
* **Resmi SAP Kimlik Şeması:** IAS hedef sisteminde küresel kullanıcı kimliği için resmi `urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid` şemasının kullanılması, sistemler arası tekil anahtar bütünlüğünü korur.
* **Dizi Ekleme (Array Append) Standardı:** S/4HANA ve BTP hedef sistemlerinde çoklu rol atamaları için `$.roles[+].value` ve `$.roleCollections[+].value` sözdiziminin `preserveArrayWithSingleElement: true` parametresiyle birlikte kullanılması RFC 7643/7644 SCIM 2.0 standardına uygundur.

> *Not: Canlıya geçiş aşamasında ve operasyonel süreçte karşılaşılabilecek teknik riskler ve bunlara yönelik zorunlu teknik önlemler [Bölüm 5.5](#55-canlıya-geçiş-day-1-ve-operasyon-day-2-mikro-teknik-risk-analizi) altında ayrıntılı bir tablo halinde paylaşılmıştır.*

---

## 2. GENİŞLETİLMİŞ TEKNİK KEŞİF VE GEREKSİNİM ANALİZ MATRİSİ

Kurumsal paydaşlarla (İK/SuccessFactors, IT Altyapı/Dizin Hizmetleri, SAP Basis/Yetki ve Siber Güvenlik) mimari netleştirme için hazırlanan 27 maddelik teknik keşif envanteri:

```
                               ┌──────────────────────────────────────────────┐
                               │       TEKNİK KEŞİF ÇALIŞTAY MATRİSİ          │
                               └──────────────────────────────────────────────┘
                                                     │
         ┌───────────────────────────────────────────┼───────────────────────────────────────────┐
         ▼                                           ▼                                           ▼
┌──────────────────┐                       ┌──────────────────┐                        ┌──────────────────┐
│   İK / SF Ekibi  │                       │   IT Altyapı     │                        │   SAP Basis      │
│ (Source of Truth)│                       │ (Entra ID / AD)  │                        │ (Hedef Sistemler)│
└──────────────────┘                       └──────────────────┘                        └──────────────────┘
```

### BÖLÜM 1: İK ve SAP SuccessFactors Ekibi (Source of Truth)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **SF-1** | Çalışan tekil anahtarı (Correlation Key) olarak hangi alan kullanılmaktadır (`personIdExternal`, `userId`, `nationalId`)? | Uçtan uca sistemler arası (AD `employeeID`, IAS `userUuid`, S/4HANA `User Alias`) eşleştirmenin temelini oluşturur. Değişmez (immutable) olmalıdır. | RFC 7643 SCIM Core Schema | Tekil alan adı ve veri tipi (Örn: 8 haneli numerik string). |
| **SF-2** | Gelecek tarihli işe alımlar (Future Hires) için süreç nasıldır? Day-1 erişimi için önceden provizyon (pre-provisioning) isteniyor mu? | Çalışanın ilk iş gününde oturum açabilmesi için AD ve SAP hesaplarının sözleşme tarihinden önce (T-3 gün) açılıp kilitli bekletilmesi gerekir. | SAP SuccessFactors EC Master Data Guide | Ön-provizyon süresi (Örn: T-3 gün oluştur, T-0 aktif et). |
| **SF-3** | Departman/Pozisyon değişimleri (Mover) SF üzerinde hangi `Event Reason` kodları ile tetiklenmektedir? | Mover senaryolarında eski yetkilerin alınıp yenilerinin tanımlanması için delta sync filtrelerinin belirlenmesi gerekir. | SAP Help: Managing Employment Lifecycle Events | Mover tetikleyici event kodları listesi (Örn: `JOB_CHANGE`, `PROMOTION`). |
| **SF-4** | İşten ayrılış (Termination / Leaver) işlemlerinde saat kuralı ve Acil Çıkış (Immediate Revocation) politikası nedir? | Standart çıkışlarda effective date gece yarısı mı devreye giriyor? Güvenlik riski durumunda anında hesap kilitlenmesi için bypass akışı var mı? | RFC 7644 SCIM Protocol | Standart çıkış saati ve acil çıkış SLA'i (Örn: Max 15 dk). |
| **SF-5** | Dış kaynak (Contractor / Vendor / Stajyer) kimlikleri SuccessFactors'te mi tutuluyor yoksa doğrudan AD'de mi açılıyor? | Eğer dış kaynaklar SF'te değilse, AD'deki hesaplarının yaşam döngüsü için ayrı bir IAM kural seti kurulmalıdır. | SAP Help: Identity Provisioning Target Systems | Dış kaynakların ana veri kaynağı (SF vs AD). |
| **SF-6** | SF ile Active Directory arasındaki mevcut entegrasyon yöntemi nedir? | Entra Inbound Provisioning API mi, özel script mi, yoksa manuel CSV export mu yapılıyor? | Microsoft Learn: Inbound Provisioning API | Entegrasyon tipi ve çalışma periyodu. |
| **SF-7** | **SuccessFactors Onboarding 2.0 modülü kullanılıyor mu? Pre-hire kullanıcıları işe başlamadan önce IAS üzerinden mi oturum açıyor?** | **13 Kasım 2026 SAP IAS zorunluluğu (KBA 2791410) kapsamında Onboarding 2.0 dış aday girişleri ilk etkilenecek alandır. Adayların henüz AD hesabı yokken IAS yerel kimliğiyle giriş yapması gerekebilir.** | **SAP KBA 2791410: Deprecation of Basic Authentication** | **Onboarding 2.0 kullanımı (Evet/Hayır) ve aday giriş yöntemi.** |

---

### BÖLÜM 2: IT Altyapı ve Dizin Hizmetleri (Active Directory / Microsoft Entra ID)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **AD-1** | Dizin mimariniz nedir (On-Premises AD, Hibrit Entra ID Connect / Cloud Sync, Pure Entra ID)? | IPS, doğrudan Entra ID Graph API (SCIM) üzerinden mi yoksa On-Prem AD üzerinden Cloud Connector / LDAP ile mi okuma yapacak? | Microsoft Graph API Users & Groups | Dizin topolojisi ve yetkili tenant/domain bilgisi. |
| **AD-2** | SF'ten gelen çalışan kimlik numarası AD'de hangi attribute'a map edilmektedir? | Çoğu kurumda `employeeID` veya `employeeNumber`, bazen de `extensionAttribute1-15` kullanılır. IPS transformation'da `sourcePath` olacaktır. | RFC 7643 Section 4.1 Enterprise User Extension | Örn: `$.employeeId` veya `$.extensionAttribute3`. |
| **AD-3** | Kullanıcı adı standardı ve UPN/E-posta ilişkisi nasıldır? | `sAMAccountName` mı `userPrincipalName` (UPN) mı esas alınacak? UPN ile `mail` her zaman birebir eşit midir? | Microsoft Learn: User Name Policies | UPN formatı ve istisna durumları. |
| **AD-4** | SAP Güvenlik Grupları için isimlendirme standardı (Naming Convention) belirlendi mi? | IPS'in AD'deki 10.000 grubu çekmesini engellemek için sadece SAP gruplarını regex ile filtrelemeliyiz. | SAP Help: Group Filtering in IPS | Örn: `SG_SAP_<SISTEM>_<ROL>_<VERI_ALANI>`. |
| **AD-5** | İç içe gruplar (Nested Groups) kullanılacak mı? Graph API rate-limit ve throttling önlemi alındı mı? | Entra ID / AD tarafında iç içe grupların çözülmesi (`expandNestedGroups`) ve filtresiz sorgular Microsoft Graph API rate-limit/throttling sınırlarına takılarak IPS senkronizasyonunu yarıda kesebilir. Sunucu taraflı filtre (`displayName startswith 'SG_SAP_'`) ve `ips.job.bulk.page.size = 100` şarttır. | Microsoft Graph API Throttling Guidance & SAP Help | Nested group kullanımı, derinlik sınırı ve filtre onay durumu. |
| **AD-6** | Bir çalışan işten ayrıldığında AD'de uygulanan standart politika nedir? | `accountEnabled = false` mı yapılıyor, kullanıcı adı önüne `DISABLED_` mı ekleniyor, gruplardan anında çıkarılıyor mu? | Microsoft Entra De-provisioning Policy | Leaver esnasında AD kullanıcı durumu ve grup aksiyonları. |
| **AD-7** | **Microsoft Entra ID tarafında P1 veya P2 lisansı var mı? "Dynamic User Groups" (Dinamik Gruplar) özelliği kullanılabilir mi?** | **P1/P2 varsa unvan, departman ve şirket koduna göre AD grupları kural bazlı dinamik dolar; IT biriminin grup ekleme/çıkarma için manuel iş yapması veya script yazması gerekmez.** | **Microsoft Learn: Dynamic Membership Rules** | **Entra ID Lisans Seviyesi (Free, P1, P2) ve dinamik grup onayı.** |

---

### BÖLÜM 3: SAP Basis & Güvenlik/Yetki Ekibi (S/4HANA, BTP, SAC)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **BASIS-1** | S/4HANA dağıtım modeli ve versiyonu nedir (Cloud Public, Private Cloud Edition, On-Premise 2022/2023)? | Dağıtım modeline göre provizyon protokolü değişir: Public Cloud ise SCIM 2.0; On-Prem/PCE ise Cloud Connector üzerinden SCIM API (`/sap/bc/scim`) veya SAP AS ABAP RFC. | SAP Help: SAP S/4HANA Target System in IPS | S/4HANA sürümü, NetWeaver platform seviyesi. |
| **BASIS-2** | S/4HANA sistemlerinde SAP Cloud Connector (SCC) kurulu ve yapılandırılmış mı? | BTP ve IPS'in on-premise ağdaki S/4HANA'ya güvenli erişebilmesi için SCC ters-tüneli şarttır. | SAP Help: Cloud Connector Overview | SCC erişilebilirliği ve subaccount bağ durumu. |
| **BASIS-3** | Principal Propagation için Kurumsal CA (X.509 Kısa Ömürlü Sertifika) altyapısı hazır mı? | Kullanıcının BTP veya Fiori Launchpad üzerinden S/4HANA'ya geçerken arka planda kendi kimliğiyle işlem yapması için X.509 istemci sertifikası üretilmesi gerekir. | SAP Help: Configuring Principal Propagation | SCC CA kurulum ve trust ilişkisi durumu. |
| **BASIS-4** | S/4HANA'da Business Partner (BP) ve Kullanıcı (SU01) senkronizasyonu nasıl yürütülmektedir? | S/4HANA'da her çalışan bir Business Partner'dır (CVI). IPS kullanıcıyı SU01'e açtığında BP eşleştirmesi otomatik mi yapılıyor yoksa HCM Employee Sync mi çalışıyor? | SAP Note 2570961: Customer Vendor Integration (CVI) | BP-User entegrasyon yöntemi (CVI senkronizasyonu). |
| **BASIS-5** | SAP BTP Subaccount mimarisi ve Global Account yapısı nasıldır? | Kaç adet subaccount var (DEV, QA, PRD)? Subaccount seviyesinde Trust Configuration'da doğrudan Entra ID mi bağlı, yoksa merkezi SAP IAS OIDC Proxy mi kullanılıyor? | SAP Help: Setting Up Trust in BTP | BTP Subaccount listesi ve mevcut Trust yapılandırması. |
| **BASIS-6** | SAP Analytics Cloud (SAC) için SCIM lisansı/API erişimi aktif mi? | SAC üzerinde kullanıcı ve Team yönetimi için SAC Custom SAML IdP ve SCIM API yetkisi olan OAuth Client hazır mı? | SAP Help: Managing Users in SAP Analytics Cloud | SAC API erişim durumu ve lisans tipi. |
| **BASIS-7** | **Kurumunuza ait kaç adet SAP Cloud Identity Services (IAS) tenant'ı mevcut?** | **SAP her müşteriye lisans kapsamında 1 Test + 1 Production IAS tenant'ı sağlar. Test ve QA ortamlarının canlıyı etkilemeden doğrulanabilmesi için ikinci tenant'ın aktif olması şarttır.** | **SAP Note 2753435: Tenant Model for IAS** | **Mevcut IAS tenant sayısı ve URL'leri.** |
| **BASIS-8** | **S/4HANA'da Firefighter (Geçici Acil Erişim) hesapları ve manuel atanan istisnai teknik roller nasıl korunacak?** | **IPS authoritative çalıştığında acil durum rollerini SU01'den silebilir; bu rollerin IPS kapsamı dışında tutulması (exclusion filter) için prefix standardı nedir?** | **SAP Help: IPS Transformation Conditions** | **Firefighter rol/kullanıcı formatı (Örn: `Z_FF_*`).** |
| **BASIS-9** | **S/4HANA Yetki Tasarımı (PFCG) ve Rol İsimlendirme Matrisi teslim süreci belirlendi mi?** | **Yetki içeriği (kilit) S/4HANA PFCG içinde bir defaya mahsus tasarlanır; IPS bu rolleri AD grupları üzerinden dağıtır (anahtar teslimi). Modül ekiplerinin tasarladıkları standart teknik rolleri (Örn: `Z_MM_BUYER_1010`) merkezi matrise teslim formatı ve SLA'i netleştirilmelidir.** | **SAP Help: PFCG Role Administration** | **Teknik rol envanter listesi ve Rol Eşleştirme Matrisi mutabakatı.** |
| **BASIS-10** | **Mevcut (Brownfield) SU01 kullanıcılarının `externalId` / Alias alanlarında personel numarası eşitlemesi yapıldı mı?** | **Mevcut S/4HANA sisteminde önceden elle açılmış kullanıcıların `SU01` Alias / `externalId` alanlarında İK personel numarası (`personIdExternal` / `employeeID`) bulunmazsa; IPS bu kullanıcıları tanıyamaz, aynı `userName` ile yeni kayıt açmaya çalışırken çakışma yaşanır ve provizyon hata verir.** | **SAP Note 2626880: SCIM User Provisioning Conflict Handling** | **Mevcut kullanıcıların Alias/externalId temizlik durumu (Hazır / Planlanacak).** |

---

### BÖLÜM 4: BTP Mimari ve Bilgi Güvenliği (Architecture & Cyber Security)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **BTP-1** | **BTP üzerinde geliştirilen uygulamalarda yetkilendirme XSUAA (Role Collection) ile mi yoksa yeni nesil AMS (Authorization Management Service) ile mi yönetilecek?** | **CAP Java/Node.js projelerinde ve BTP Cloud Foundry/Kyma ortamlarında XSUAA standarttır; ancak yeni mimarilerde AMS kullanımı BTP token yapısını ve rol hiyerarşisini doğrudan etkiler.** | **SAP Help: Authorization and Trust Management Service** | **Tercih edilen yetki motoru (XSUAA vs AMS).** |
| **SEC-1** | **MFA (Çok Faktörlü Doğrulama) tetiklemesi nerede yapılacak (Entra ID Conditional Access mi, yoksa SAP IAS Risk-Based Authentication mı)?** | **Her iki tarafta da MFA açılırsa çalışanlar çift faktör döngüsüne (Double-MFA) girerek kilitlenir. Mimari ilke olarak MFA kurumsal IdP olan Entra ID'de işletilmeli; IAS pass-through/proxy çalışmalıdır.** | **NIST SP 800-63B: Digital Identity Guidelines** | **MFA politikası ve yetkili karar mercii.** |
| **SEC-2** | **Görevler Ayrılığı (SoD) ve Yetki Çakışması denetimi nasıl sağlanacak? SAP IAG (Cloud Identity Access Governance) entegrasyonu planlanıyor mu?** | **IPS körlemesine grup basarsa finansal denetimlerde (SOX/BDDK) "Satıcı Açan Kişinin Fatura Onaylaması" gibi SoD açıkları oluşur. Kritik roller için IPS öncesi bir onay/GRC katmanı gerekip gerekmediği netleşmelidir.** | **SAP Help: SAP Cloud Identity Access Governance** | **SoD risk kontrol mekanizması (Manuel vs SAP IAG/GRC).** |

---

## 3. İK YAŞAM DÖNGÜSÜNDEKİ UÇ SENARYOLAR (JML EDGE CASES)

Kurumsal operasyonlarda sistemi en sık sekteye uğratan 4 kritik uç senaryo ve teknik çözüm mimarisi:

### 1. Çift Şapkalılık / Eşzamanlı İstihdam (Concurrent Employment)
* **Problem:** Bir çalışan hem Türkiye şirketinde Finans Direktörü, hem de Hollanda iştirakinde Yönetim Kurulu Üyesidir. SuccessFactors'te tek bir `personIdExternal` altında iki farklı `userId` ve iki farklı istihdam sözleşmesi bulunur. Ancak çalışanın kurumsal Active Directory'de yalnızca **tek bir hesabı** (`sAMAccountName`) vardır.
* **Mimari Çözüm:** 
  * Entra ID üzerinde kullanıcıya her iki görevi temsil eden güvenlik grupları (`SG_SAP_S4H_TR_FIN_DIR` ve `SG_SAP_S4H_NL_BOARD`) atanır.
  * BTP ve S/4HANA tarafında yetkiler **kümülatif (birleştirilmiş)** olarak yüklenir.
  * Eğer S/4HANA tarafında yasal/denetim gereği iki görevin kesinlikle ayrı kullanıcı kimlikleriyle (`AHMET_TR` ve `AHMET_NL`) yapılması zorunluysa; IT birimi AD tarafında ikinci bir teknik hesap açmalı ve bu hesap SF'teki ilgili `userId` ile ilişkilendirilmelidir.

### 2. Tekrar İşe Alım (Rehire)
* **Problem:** 2 yıl önce şirketten ayrılmış bir personel tekrar işe başladığında sistemlerin çakışması ve eski yetkilerin canlanması riski.
* **Mimari Çözüm:**
  * SuccessFactors çalışanın eski `personIdExternal` anahtarını korur.
  * AD tarafında eski hesap yeniden aktive edilir (`accountEnabled = true`).
  * IPS, S/4HANA'da kilitli bekleyen eski `SU01` hesabını `externalId` eşleşmesi sayesinde bulur. Yeni bir kullanıcı açmaya çalışıp hata vermez; mevcut hesabı **Un-lock** eder ve eski departmanından kalan tüm bayat rolleri silip sadece yeni AD gruplarını yazar.

### 3. Şirket İçi Yatay Geçiş ve Çift Yetki Toleransı (Cross-Company Transfer)
* **Problem:** Çalışan 1 Kasım itibarıyla Satınalma'dan Finans'a geçmektedir; ancak ay sonu kapanışlarını tamamlayabilmesi için 15 Kasım'a kadar her iki şirkette/departmanda da işlem yapabilmesi gerekir.
* **Mimari Çözüm:**
  * AD tarafında **"Grace Period" (Tolerans Süresi)** uygulanır. Çalışan 15 gün boyunca her iki güvenlik grubunda da tutulur.
  * 15. günün sonunda IT otomasyonu eski departman grubunu siler. IPS'in bir sonraki artımlı senkronizasyonunda eski roller S/4HANA'dan otomatik olarak düşer.

### 4. İsim / Soyisim Değişikliği (Name/Surname Change)
* **Problem:** Evlilik veya yasal nedenlerle çalışanın soyadı değişir. AD'de `userPrincipalName` ve `mail` güncellenir (`ayse.kaya` $\rightarrow$ `ayse.yilmaz`).
* **Mimari Çözüm:**
  * Uçtan uca sistemler birbirine `mail` veya `userName` ile değil; **değişmez (immutable) `employeeId` / `userUuid`** ile bağlıdır.
  * IPS, S/4HANA'daki kaydı `externalId` ile bulduğu için kullanıcının kullanıcı adı veya e-postası değişse bile yetki ilişkisi kopmaz; profil bilgisi sorunsuz güncellenir.

---

## 4. ROL VE YETKİ MİMARİSİ: İŞ BÖLÜMÜ (AUTHORING VS. PROVISIONING) VE VERİ KISITI

### 4.1. MİMARİ İŞ BÖLÜMÜ: YETKİ TASARIMI (MODÜL İÇİ) VS. YETKİ DAĞITIMI (BTP / IPS MERKEZİ)

Teknik tarafta paydaş sorumluluklarını netleştirmek adına şu ayrımı yapmak faydalı olacaktır: Biz yetki tasarlamayacağız, S/4HANA tarafındaki yetkileri danışmanlar PFCG içinde hazırlayacak. Biz sadece bu rollerin teknik isimlerini alıp Active Directory gruplarına bağlayacağız. Böylece modül içine girip tek tek elle rol atama işi ortadan kalkacak. Fonksiyonel yetkilerin kapsamı modüllerin kendi içinde belirlenir; bu yetkilerin kullanıcılara atanması ise BTP/IPS üzerinden merkezi olarak yönetilir. BTP veya IPS tek başına modüllerin içine girmeden fonksiyonel bir yetki oluşturamaz.

```
┌────────────────────────────────────────────────────────┐
│  AŞAMA 1: YETKİ TASARIMI (MODÜL İÇİ TANIMLAMA / PFCG)  │
│  • S/4HANA: PFCG işlem koduyla İş Rolü oluşturulur.    │
│  • SAC: Roller ve takımlar (Teams) tanımlanır.         │
│  • BTP CAP: xs-security.json içinde scopelar yazılır.  │
│  ==> Çıktı: Teknik Rol Adı (Örn: Z_MM_BUYER_1010)      │
└──────────────────────────┬─────────────────────────────┘
                           │ (Teknik rol adları matrise verilir)
                           ▼
┌────────────────────────────────────────────────────────┐
│  AŞAMA 2: YETKİ DAĞITIMI (BTP / SAP IPS - OTOMASYON)   │
│  • AD Grubu (SG_SAP_*) <──> Teknik Rol eşleşmesi       │
│  • Kullanıcı işe girdiğinde/değiştiğinde IPS rolü yazar│
│  • Modüllere kullanıcı açmak için bir daha girilmez!   │
└────────────────────────────────────────────────────────┘
```

#### Rol Dağılımı ve Sorumluluk Sınırları (RACI Modeli)

| Aşama / Sorumluluk | Yapılan İşlem | İşlemin Yapıldığı Yer | BTP / IPS Üzerinden Yapılabilir mi? | Sorumlu Ekip |
| :--- | :--- | :--- | :---: | :--- |
| **1. Yetki Tasarımı (Authoring)** | "Satın Alma Onayı hangi T-Code ve tabloları kapsar?" sorusunun cevabı tanımlanır. | **Modülün Kendi İçi** (S/4HANA `PFCG`, SAC, Ariba, BTP CAP `xs-security.json`) | **HAYIR** | Modül Danışmanları & Yetki Ekibi |
| **2. Rolün İsimlendirilmesi** | Role standart teknik isim verilir (Örn: `Z_BR_GL_ACCOUNTANT_1000`). | **Modülün Kendi İçi** | **HAYIR** | Modül Yetki Ekibi |
| **3. Rol Eşleme (Mapping)** | AD Güvenlik Grubu ile modüldeki teknik rol adı eşleştirilir. | **Rol Eşleştirme Matrisi & IPS Transformations** | **EVET** | Dijital Dönüşüm & IAM Ekibi |
| **4. Rolün Atanması (Provisioning)** | İK/AD'den gelen çalışana rolün otomatik verilmesi ve geri alınması. | **SAP IPS & BTP Role Collection Mapping** | **EVET** | Tam Otomasyon (Zero-Touch) |

> **Temel Mimari Yaklaşım:** Modül danışmanları standart iş rollerini modülün içinde tanımlayıp teknik isimlerini teslim ettikten sonra, çalışanlara yetki vermek için modüllerin içine elle girilmez; atama ve geri alma işlemleri kurumsal kimlik akışıyla otomatik yürütülür.

---

### 4.2. ROL EŞLEŞTİRME VE "VERİ KISITI" (ABAC VS. DERIVED ROLES) MİMARİSİ

Standard SCIM 2.0 protokolü ve SAP IPS, S/4HANA içerisindeki bir PFCG rolünün organizasyonel seviyelerini (Şirket Kodu: 1000, Satın Alma Örgütü: 1010 vb.) satır satır enjekte edemez. Bu kısıt aşağıdaki 2 yöntemle mimari güvenceye alınır:

```
[İş İhtiyacı: Muhasebe Rolü + Şirket Kodu 1000 Kısıtı]
                          │
         ┌────────────────┴────────────────┐
         ▼                                 ▼
[S/4HANA Arka Uç]                 [BTP CAP Uygulamaları]
Türetilmiş Roller                 Runtime JWT Claim (ABAC)
(Derived Roles Modeli)            (CDS Declarative Security)
Örn: Z_BR_GL_ACCOUNTANT_1000      @restrict: [{ where: 'companyCode = $user.CompanyCode' }]
```

#### A. S/4HANA için Türetilmiş Roller (Derived Roles) Modeli
S/4HANA yetki ekibi organizasyonel alanları boş olan ana roller yerine, ilgili veri kısıtları önceden tanımlanmış **Türetilmiş Roller** üretir:
* `SG_SAP_S4H_FI_GL_1000`  -->  `Z_BR_GL_ACCOUNTANT_1000` (Company Code: 1000)
* `SG_SAP_S4H_FI_GL_2000`  -->  `Z_BR_GL_ACCOUNTANT_2000` (Company Code: 2000)
* `SG_SAP_S4H_MM_BUYER_1010` --> `Z_BR_PURCHASER_1010` (Purchasing Org: 1010)

#### B. BTP CAP Uygulamaları için Dinamik Nitelik (ABAC) Modeli
BTP uygulamalarında roller statiktir (`FinancialAnalyst`), veri kısıtı ise dinamiktir:
1. Entra ID'de kullanıcının profilindeki `companyCode` veya `costCenter` attribute'u okunur.
2. IAS üzerinden JWT ID Token içine custom attribute olarak basılır.
3. BTP Java/Node.js CAP servis katmanında (`service.cds`) deklaratif kural işletilir:
   ```cds
   annotate CatalogService.Invoices with @(
       restrict: [
           { grant: 'READ', to: 'FinancialAnalyst', where: 'companyCode = $user.companyCode' }
       ]
   );
   ```

---

## 5. TAKTİKSEL OPERASYON KATMANI: YAPILABİLİRLİK MATRİSİ, PAYDAŞ ÖN KOŞULLARI VE SINIR KILAVUZU

Taktiksel katman, sistemin teorik vizyonuyla kod seviyesindeki JSON yapılandırması arasında kalan operasyonel köprüdür: **"Hangi senaryo gerçekten çalışır, bunu çalıştırmak için hangi somut şartlar gerekir ve nerede sınır çizeceğiz?"**

### 5.1. Taktiksel Senaryo ve Yapılabilirlik Matrisi

| Senaryo | Yapılabilirlik Derecesi | Sahada Nasıl İşler? | Gerektirdiği Asgari Şartlar | Sınırı / Dikkat Edilecek Nokta |
|:---|:---|:---|:---|:---|
| **1. Standart İşe Giriş (Day-1 Birthright)** | **Tam (%100)** | İK SF'te işe alımı onaylar $\rightarrow$ IT AD'de hesabı ve `SG_SAP_*` grubunu açar $\rightarrow$ IPS kullanıcıyı S/4HANA'da açıp rolü bağlar; BTP'de oturum anında rol koleksiyonu eşleşir. | • SF $\leftrightarrow$ AD entegrasyonunun çalışması.<br>• Hedef sistemde standart PFCG veya BTP rolünün önceden tanımlı olması. | Kullanıcı S/4HANA'da hemen açılsa bile Fiori tarafında işlem yapabilmesi için arka plandaki Business Partner (BP/CVI) senkronizasyonunun tamamlanması gerekir. |
| **2. İşten Ayrılış (Leaver - Erişim Kesme)** | **Tam (%100)** | AD hesabı devre dışı kaldığında (`accountEnabled = false`), IPS hedef sistemlerdeki kullanıcıları kilitler (`active = false`). | • AD'deki durum bilgisinin IPS'e doğru map edilmesi (`$.accountEnabled` $\rightarrow$ `active`). | Standart IPS periyodu (örneğin 30-60 dk) aralığında çalışır. "Acil çıkış" gerekiyorsa ilk 15 dakika için S/4HANA `SU01` ve BTP üzerinden manuel kilitleme bypass akışı tanımlanmalıdır. |
| **3. Departman / Görev Değişimi (Mover)** | **Yüksek (%90)** | SF'te pozisyon değişir $\rightarrow$ AD grubu değişir $\rightarrow$ IPS eski yetkileri geri alır, yeni departmanın rollerini tanımlar. | • AD grubundan çıkarma işleminin zamanında yapılması.<br>• IPS'te ilgili hedef sistemin yetki güncelleme modunun açık olması. | Eski işlerin devri için 10-15 günlük bir yetki çakışması (çift yetki toleransı) gerekiyorsa, AD tarafında eski grubun silinmesi bu süre kadar geciktirilmelidir (Grace Period). |
| **4. Organizasyonel Veri Kısıtı (Şirket Kodu / Masraf Merkezi)** | **Koşullu (%60)** | Kullanıcının sadece kendi şirket kodunu (Örn: `1000`) görmesi ihtiyacı. | • **S/4HANA için:** Türetilmiş Roller (Derived Roles) şarttır.<br>• **BTP için:** AD/IAS profilindeki şirket kodu attribute'unun JWT token'a basılması gerekir. | IPS doğrudan PFCG içindeki organizasyon seviyelerini dolduramaz. S/4HANA yetki ekibi `Z_FI_ACCOUNTANT_1000` gibi türetilmiş rolleri hazır etmezse bu senaryo işletilemez. |
| **5. Dış Kaynak / Danışman / Stajyer Yönetimi** | **Yüksek (%85)** | SF'te kaydı olmayan dış kullanıcıların SAP erişimlerinin yönetilmesi. | • IT'nin bu kişilere AD tarafında son kullanma tarihli (Account Expiration Date) hesap açması.<br>• İlgili SAP güvenlik grubuna eklenmesi. | İK süreci devrede olmadığı için bu hesapların yaşam döngüsü sorumluluğu doğrudan IT Altyapı ve ilgili sözleşme yöneticisinde kalır. |
| **6. Geçici Görev / Acil Durum Erişimi (Vekalet)** | **Hibrit (%50)** | Bir personelin 1 haftalığına başka bir birimin onay ekranına bakması. | • AD tarafında süreli grup üyeliği (veya PIM) **VEYA**<br>• S/4HANA tarafında manuel atama + Firefighter filtresi. | IPS'te `authoritative = true` ise elle verilen geçici roller silinir. Bu rollerin başına `Z_EXC_*` veya `Z_FF_*` konularak IPS kapsamı dışına çıkarılması gerekir. |

---

### 5.2. Taktiksel Ön Koşullar: Hangi Ekipten Asgari Ne İstenecek?

Projeyi bürokrasiye boğmadan, her birimden masaya getirmesi beklenen net "teslimat kalemleri":

```
┌────────────────────────┐      ┌────────────────────────┐      ┌────────────────────────┐
│     İK / SF EKİBİ      │      │     IT / AD EKİBİ      │      │    SAP BASIS / YETKİ   │
├────────────────────────┤      ├────────────────────────┤      ├────────────────────────┤
│ 1. Değişmez Personel No│      │ 1. Sabit Grup Formatı  │      │ 1. Rol İsim Listesi    │
│ 2. Standart Pozisyon   │      │ 2. employeeID Eşleme   │      │ 2. SCC / SCIM Tüneli   │
│ 3. Temiz Event Kodları │      │ 3. Servis Kimliği (API)│      │ 3. İstisna Regex Kuralı│
└────────────────────────┘      └────────────────────────┘      └────────────────────────┘
```

#### A. İK / SuccessFactors Ekibinden İstenecekler

1. **Değişmez Personel Numarası (`personIdExternal`):** TC Kimlik veya isim değil; hiçbir koşulda değişmeyen tekil anahtarın ana kaynak olarak teyit edilmesi.
2. **Standart Pozisyon Tanımları:** Unvanların serbest metin yerine sistemdeki standart pozisyon kodlarıyla (`JobCode` / `Position ID`) ilişkilendirilmiş olması.
3. **Mover / Leaver Olay Kodları:** Hangi statü değişiminin (Örn: `TER`, `REH`, `XFR`) sistemde kullanıcı kilitlenmesine veya grup değişimine yol açacağının listesi.

#### B. IT / Dizin Hizmetleri (Active Directory / Entra ID) Ekibinden İstenecekler

1. **Grup İsimlendirme Standardı:** SAP gruplarının diğer kurumsal gruplarla karışmaması için kesin bir regex standardı (Örn: `SG_SAP_<SISTEM>_<ROL>`).
2. **Öznitelik (Attribute) Eşleme:** SF'ten gelen personel numarasının AD üzerinde `employeeID` alanına yazıldığının garantilenmesi.
3. **IPS Servis Kaydı (App Registration):** IPS'in Entra ID Graph API'den sadece bu grupları okuyabilmesi için `User.Read.All` ve `GroupMember.Read.All` yetkilerine sahip bir Client ID ve Secret/Sertifika üretilmesi.
4. **Graph API Throttling Koruması ve Filtreleme:** Kurumsal dizindeki binlerce grubun filtresiz çekilerek Microsoft Graph API rate-limit/throttling sınırlarına takılmaması için `entra.id.filter.groups: displayName startswith 'SG_SAP_'` sunucu taraflı filtresinin ve `ips.job.bulk.page.size = 100` sınırının konfigüre edilmesi.

#### C. SAP Basis ve Yetki Danışmanlarından İstenecekler

1. **Standart Teknik Rol Envanteri:** Modül danışmanlarının S/4HANA tarafında tasarladığı rollerin kesin teknik isimleri (Örn: `Z_BR_PURCHASER_1010`).
2. **Cloud Connector ve SCIM Tüneli:** BTP ile S/4HANA arasındaki Cloud Connector üzerinde `/sap/bc/scim` yolunun ve servisinin (`SICF`) aktif edilmesi.
3. **İstisna Öneki (Exclusion Prefix):** Manuel veya acil durumlarda verilecek roller için bir ön ek üzerinde mutabakat sağlanması (Örn: `Z_FF_*`, `Z_EXC_*`).
4. **Mevcut (Brownfield) SU01 Kullanıcı Temizliği:** Canlıya geçiş öncesinde mevcut aktif kullanıcıların `SU01` Alias / `externalId` alanlarına İK personel numarasının (`employeeID`) toplu yazılarak SCIM çakışmalarının önlenmesi.

---

### 5.3. Sahadaki Operasyonel Tıkanma Noktaları ve Pratik Çözümleri

* **Tıkanma 1: AD Grup Enflasyonu (Grup Patlaması)**
  * *Semptom:* Her unvan ve her şirket kodu için ayrı grup açılmaya başlandığında AD tarafında yüzlerce `SG_SAP_*` grubu birikir, IT yönetemez hale gelir.
  * *Taktik Çözüm:* Grupları fonksiyonel düzeyde tutmak. Örneğin sadece `SG_SAP_S4H_SATINALMA` grubu açılır; şirket kodu kısıtı ise kullanıcının AD profilindeki departman/şirket kodu özniteliğinden (attribute) dinamik beslenir.

* **Tıkanma 2: S/4HANA CVI / Business Partner Gecikmesi**
  * *Semptom:* IPS kullanıcıyı S/4HANA `SU01`'e yazar, kullanıcı oturum açabilir ama Fiori ekranında fatura veya onay işlemine bastığında "İş Ortağı Bulunamadı" hatası alır.
  * *Taktik Çözüm:* S/4HANA HCM $\leftrightarrow$ Business Partner senkronizasyon arka plan işinin (`/BOM/` senkronizasyon job'ı) IPS tetikleme periyodundan daha sık veya hemen ardışık çalışacak şekilde zamanlanması gerekir.

* **Tıkanma 3: "Authoritative" Senkronizasyon Kaygısı**
  * *Semptom:* Proje ekibi, otomasyon açıldığında mevcut çalışanların yetkilerinin silinmesinden korktuğu için canlıya geçiş onayını geciktirir.
  * *Taktik Çözüm:* Canlıya geçişte `s4hana.authoritative.role.management` parametresi `false` olarak ayarlanır. Sistem sadece yeni rolleri ekler; içeride elle verilmiş mevcut rollere dokunmaz. Böylece hiçbir çalışanın ekranı kapanmaz, operasyonel risk sıfırlanır.

---

### 5.4. Taktiksel Sınır Çizme Kılavuzu (Neyi Otomatikleştirelim, Neyi Bırakalım?)

Kurulumun tıkanmaması için çizilmesi gereken operasyonel sınırlar:

```
                            [ YETKİ TALEBİ / İHTİYAÇ ]
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
       [ Standart / Pozisyona Bağlı? ]             [ İstisnai / Riskli / Geçici? ]
                    │                                       │
                    ▼                                       ▼
       ┌─────────────────────────┐             ┌─────────────────────────┐
       │   TAM OTOMASYON (IPS)   │             │    MANUEL / İŞ AKIŞI    │
       ├─────────────────────────┤             ├─────────────────────────┤
       │ • Fiori ESS / İzin      │             │ • Finansal ödeme onayı  │
       │ • Departman ana ekranı  │             │ • Satıcı ana verisi     │
       │ • BTP kurumsal portal   │             │ • Proje bazlı geçici rol│
       └─────────────────────────┘             └─────────────────────────┘
```

* **Otomasyona Alınacaklar (%80 Hacim):** Tüm yetkileri ilk günden otomatikleştirmek yerine; standart işe giriş ekranları (izin, bordro, departman ana ekranı) otomatik gelsin. Fiori ESS/MSS, genel kurumsal portallar ve standart departman ekranları AD grupları üzerinden otomatik atanabilir.
* **Manuel / Süreçte Bırakılacaklar (%20 Hacim):** Riskli finansal onaylar veya özel yetkiler için yine eskisi gibi onay formu işletilsin, böylece denetim riski oluşmasın. Çift onay gerektiren yüksek limitli ödeme işlemleri, satıcı ana verisi ve sistem parametre rolleri için mevcut onay akışları korunabilir veya ileride kurulacak SAP IAG modülü değerlendirilebilir.

---

### 5.5. Canlıya Geçiş (Day-1) ve Operasyon (Day-2) Mikro Teknik Risk Analizi

Canlıya geçiş anında ve takip eden operasyonel süreçte sistemleri kilitleyebilecek 6 kritik mikro teknik risk ve sahada uygulanması gereken önlemler aşağıda analiz edilmiştir:

| Risk Alanı | Teknik Kök Neden ve Semptom | Operasyonel Etki (Day-1 / Day-2) | Zorunlu Teknik Önlem / Çözüm |
|:---|:---|:---|:---|
| **1. Brownfield (Mevcut) SU01 Kullanıcılarının Eşleşmemesi** | Mevcut S/4HANA sisteminde önceden elle açılmış kullanıcıların `SU01` Alias / `externalId` alanlarında personel numarasının (`personIdExternal`) bulunmaması. | IPS senkronizasyonu çalıştığında kullanıcıyı `externalId` üzerinden bulamaz; aynı `userName` ile yeni kayıt açmaya çalışırken çakışma yaşanır ve provizyon hata verir. | Canlıya geçiş öncesinde SAP Basis ekibi mevcut aktif `SU01` kullanıcılarının `externalId` / `Alias` alanlarını İK personel numarasıyla (`employeeID`) eşitlemelidir. |
| **2. S/4HANA CVI ve Business Partner (BP) Senkronizasyon Gecikmesi** | S/4HANA mimarisinde her çalışanın aynı zamanda bir İş Ortağı (Business Partner - BP) olması zorunludur (SAP Note 2570961). | IPS, `SU01` hesabını ve PFCG rolünü saniyeler içinde açar; ancak arka plandaki HCM $\leftrightarrow$ BP (`/BOM/`) senkronizasyon işi çalışmadıysa kullanıcı Fiori ekranlarında "İş Ortağı Bulunamadı" hatası alır. | S/4HANA üzerindeki `/BOM/` (CVI/HCM Employee Sync) arka plan işi, IPS çalışma periyodundan daha sık veya hemen ardışık çalışacak şekilde zamanlanmalıdır. |
| **3. Entra ID Graph API Rate-Limit / Throttling ve Nested Groups** | Kurumsal dizindeki binlerce grubun filtresiz çekilmesi veya iç içe grupların (`expandNestedGroups`) derinlemesine çözümlenmesi. | IPS senkronizasyon süresi uzar, ağ trafiği şişer ve Microsoft Graph API üzerinde paket limitlerine takılarak senkronizasyon yarıda kesilir. | Kaynak sistem özelliklerinde `entra.id.filter.groups: displayName startswith 'SG_SAP_'` sunucu taraflı filtresi ve `ips.job.bulk.page.size = 100` sınırı zorunlu tutulmalıdır. |
| **4. S/4HANA Sözleşmesel Lisans (FUE) Tipi Aşımı** | IPS üzerinden açılan kullanıcılara `SU01` üzerinde varsayılan olarak yüksek maliyetli kullanıcı tipi (*Professional*) atanması. | Sadece izin talebi (ESS) veya temel rapor kullanacak personelin tam lisans havuzunu tüketmesi sonucu sözleşmesel FUE (Full User Equivalent) maliyeti aşılır. | S/4HANA tarafında varsayılan sözleşmesel kullanıcı tipi (*Professional*, *Functional*, *Productivity*) modül yetki ekibiyle önceden mutabık kalınarak sınırlandırılmalıdır. |
| **5. Koruyucu Modda Yetki Kayması (Continuous Drift)** | Geçiş fazında mevcut manuel rolleri korumak için `s4hana.authoritative.role.management = false` parametresinin kullanılması. | `authoritative = false` iken `SU01` üzerinden el altından verilen roller IPS tarafından silinmez; zamanla Entra ID grupları ile S/4HANA gerçek yetkileri birbirinden kopar. | Sistem kararlı hale geldikten sonra ayda bir kez IPS dışından elle atanmış yetkilerin denetim raporu alınmalı ve temizlik sonrası kademeli olarak `true` moduna geçilmelidir. |
| **6. Zamanlanmış Senkronizasyon Gecikmesi ve Acil Çıkışlar** | IPS zamanlanmış (scheduled) bir SCIM ETL motoru olarak belirli periyotlarla (örn. 30-60 dk) çalışır. | Güvenlik riski taşıyan acil işten çıkışlarda (Immediate Revocation) bir sonraki IPS döngüsünü beklemek 15 dakikalık SLA hedefini ihlal edebilir. | Acil çıkış durumlarında ilk 15 dakika içinde oturumu sonlandırmak için S/4HANA `SU01` ve BTP üzerinden manuel kilitleme bypass prosedürü tanımlanmalıdır. |

---

## 6. DOĞRULANMIŞ SAP IPS JSON KOD SPESİFİKASYONU

Aşağıdaki dönüşüm kodları; SCIM 2.0 sözdizimi, SAP Identity Directory resmi şemaları, Delta Sync değişkenleri ve Firefighter koruma filtreleriyle tam uyumludur.

### A. Kaynak Sistem: Microsoft Entra ID (Source System)

```json
{
  "user": {
    "mappings": [
      {
        "sourcePath": "$.id",
        "targetVariable": "entityIdExternal"
      },
      {
        "sourcePath": "$.id",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.userPrincipalName",
        "targetPath": "$.userName"
      },
      {
        "sourcePath": "$.mail",
        "targetPath": "$.emails[0].value",
        "optional": true
      },
      {
        "sourcePath": "$.givenName",
        "targetPath": "$.name.givenName",
        "optional": true
      },
      {
        "sourcePath": "$.surname",
        "targetPath": "$.name.familyName",
        "optional": true
      },
      {
        "sourcePath": "$.displayName",
        "targetPath": "$.displayName"
      },
      {
        "sourcePath": "$.employeeId",
        "targetPath": "$.externalId",
        "explanation": "Correlation ID: SuccessFactors ve SAP dünyasını bağlayan tekil kök anahtar."
      },
      {
        "sourcePath": "$.accountEnabled",
        "targetPath": "$.active",
        "type": "boolean"
      },
      {
        "sourcePath": "$.companyName",
        "targetPath": "$.urn:ietf:params:scim:schemas:extension:enterprise:2.0:User.company",
        "optional": true
      },
      {
        "sourcePath": "$.memberOf",
        "targetPath": "$.groups",
        "preserveArrayWithSingleElement": true,
        "optional": true
      }
    ]
  },
  "group": {
    "mappings": [
      {
        "sourcePath": "$.id",
        "targetVariable": "groupIdExternal"
      },
      {
        "sourcePath": "$.id",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.displayName",
        "targetPath": "$.displayName",
        "condition": "$.displayName =~ /SG_SAP_.*/",
        "explanation": "Yalnızca SG_SAP_ ile başlayan güvenlik grupları IPS'e çekilir."
      },
      {
        "sourcePath": "$.members",
        "targetPath": "$.members",
        "preserveArrayWithSingleElement": true,
        "optional": true
      }
    ]
  }
}
```

---

### B. Hedef Sistem: SAP Cloud Identity Services - IAS (Identity Directory SCIM 2.0)

```json
{
  "user": {
    "mappings": [
      {
        "sourceVariable": "entityIdTargetSystem",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.userName",
        "targetPath": "$.userName"
      },
      {
        "sourcePath": "$.externalId",
        "targetPath": "$.externalId"
      },
      {
        "sourcePath": "$.externalId",
        "targetPath": "$.urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid",
        "explanation": "Resmi SAP Identity Directory Global User ID alanı."
      },
      {
        "sourcePath": "$.emails[0].value",
        "targetPath": "$.emails[0].value"
      },
      {
        "sourcePath": "$.name.givenName",
        "targetPath": "$.name.givenName"
      },
      {
        "sourcePath": "$.name.familyName",
        "targetPath": "$.name.familyName"
      },
      {
        "sourcePath": "$.displayName",
        "targetPath": "$.displayName"
      },
      {
        "sourcePath": "$.active",
        "targetPath": "$.active"
      },
      {
        "sourcePath": "$.urn:ietf:params:scim:schemas:extension:enterprise:2.0:User.company",
        "targetPath": "$.urn:ietf:params:scim:schemas:extension:enterprise:2.0:User:company",
        "optional": true
      },
      {
        "sourcePath": "$.groups[?(@.value =~ /SG_SAP_.*/)]",
        "targetPath": "$.groups",
        "preserveArrayWithSingleElement": true,
        "optional": true
      }
    ]
  },
  "group": {
    "mappings": [
      {
        "sourceVariable": "entityIdTargetSystem",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.displayName",
        "targetPath": "$.displayName",
        "condition": "$.displayName =~ /SG_SAP_.*/"
      },
      {
        "sourcePath": "$.members",
        "targetPath": "$.members",
        "preserveArrayWithSingleElement": true,
        "optional": true
      }
    ]
  }
}
```

---

### C. Hedef Sistem: SAP S/4HANA (SCIM 2.0 / Cloud Connector)

```json
{
  "user": {
    "mappings": [
      {
        "sourceVariable": "entityIdTargetSystem",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.userName",
        "targetPath": "$.userName"
      },
      {
        "sourcePath": "$.externalId",
        "targetPath": "$.externalId",
        "explanation": "S/4HANA SU01 Personel No veya Alias alanı."
      },
      {
        "sourcePath": "$.emails[0].value",
        "targetPath": "$.emails[0].value",
        "optional": true
      },
      {
        "sourcePath": "$.name.givenName",
        "targetPath": "$.name.givenName"
      },
      {
        "sourcePath": "$.name.familyName",
        "targetPath": "$.name.familyName"
      },
      {
        "sourcePath": "$.active",
        "targetPath": "$.active",
        "explanation": "Kullanıcı ayrıldığında SU01 kilidi devreye girer."
      },
      {
        "condition": "('SG_SAP_S4H_FI_GL_1000' IN $.groups[*].display)",
        "constant": "Z_BR_GL_ACCOUNTANT_1000",
        "targetPath": "$.roles[+].value",
        "preserveArrayWithSingleElement": true,
        "explanation": "Valid SCIM Append Sözdizimi: Genel Muhasebe Şirket 1000 Türetilmiş Rolü."
      },
      {
        "condition": "('SG_SAP_S4H_MM_BUYER_1010' IN $.groups[*].display)",
        "constant": "Z_BR_PURCHASER_1010",
        "targetPath": "$.roles[+].value",
        "preserveArrayWithSingleElement": true,
        "explanation": "Valid SCIM Append Sözdizimi: Satınalma Örgütü 1010 Türetilmiş Rolü."
      },
      {
        "condition": "('SG_SAP_COMMON_ESS' IN $.groups[*].display)",
        "constant": "SAP_BR_EMPLOYEE",
        "targetPath": "$.roles[+].value",
        "preserveArrayWithSingleElement": true,
        "explanation": "Tüm çalışanlar için ESS temel Fiori rolü."
      }
    ]
  }
}
```

---

### D. Hedef Sistem: SAP BTP Subaccount (XSUAA / Role Collection Target)

```json
{
  "user": {
    "mappings": [
      {
        "sourceVariable": "entityIdTargetSystem",
        "targetVariable": "entityIdTargetSystem"
      },
      {
        "sourcePath": "$.userName",
        "targetPath": "$.userName"
      },
      {
        "sourcePath": "$.emails[0].value",
        "targetPath": "$.emails[0].value"
      },
      {
        "sourcePath": "$.name.givenName",
        "targetPath": "$.name.givenName"
      },
      {
        "sourcePath": "$.name.familyName",
        "targetPath": "$.name.familyName"
      },
      {
        "sourcePath": "$.externalId",
        "targetPath": "$.originKey",
        "optional": true
      },
      {
        "condition": "('SG_SAP_BTP_ADMINS' IN $.groups[*].display)",
        "constant": "Subaccount Administrator",
        "targetPath": "$.roleCollections[+].value",
        "preserveArrayWithSingleElement": true,
        "explanation": "BTP Subaccount Yönetici Rol Koleksiyonu Ataması."
      }
    ]
  }
}
```

---

## 7. UÇTAN UCA TEKNİK KURULUM, ENTEGRASYON VE DOĞRULAMA KILAVUZU (5 FAZLI REHBER)

SAP ekosisteminde kimlik federasyonu (IAS), kullanıcı/rol provizyonu (IPS) ve yetkilendirme (BTP & S/4HANA) mekanizmalarının uçtan uca devreye alınması; ağ tünellerinden sertifika zincirlerine, SCIM dönüşümlerinden çalışma zamanı güven ilişkilerine kadar 5 ana kurulum fazından oluşur.

### FAZ 1: Altyapı, Ağ Tüneli ve Ön Hazırlık Kurulumu

Kurulumun bu aşamasında sistemlerin birbirine güvenli erişimini sağlayacak ağ katmanı, tüneller ve dizin izinleri yapılandırılır:

```
[On-Premise / PCE Ağ]                 [Güvenli Tünel]                [SAP BTP Bulut Ağı]
S/4HANA (ICF / RFC)  ◄──────►  SAP Cloud Connector (SCC)  ◄──────►  BTP Subaccount / IPS Engine
```

#### 1.1. Microsoft Entra ID Uygulama Kaydı (App Registration)

SAP IPS'in Entra ID Graph API üzerinden kullanıcı ve grupları okuyabilmesi için kurumsal dizinde bir servis kimliği açılır:

1. **App Registration:** Entra ID yönetim portalından `SAP-IPS-Inbound-Connector` adıyla yeni bir uygulama kaydedilir.
2. **API İzinleri (Application Permissions):**
   * `User.Read.All` (Kullanıcı profillerini ve `employeeId` değerini okuma).
   * `GroupMember.Read.All` (Yalnızca güvenlik gruplarını ve grup üyelerini okuma).
3. **Kimlik Doğrulama:** Bir **Client Secret** oluşturulur veya kurumsal PKI altyapısından üretilen bir **X.509 Sertifikası** yüklenir (Önerilen: Sertifika tabanlı kimlik doğrulama).
4. **Çıktılar:** `Tenant ID`, `Client ID` ve `Client Secret / Certificate` değerleri IPS yapılandırması için not edilir.

#### 1.2. SAP Cloud Connector (SCC) Kurulumu ve Subaccount Bağlantısı

On-Premise veya Private Cloud S/4HANA sistemlerinin buluta açılması için DMZ veya iç ağda kurulu SCC üzerinde tünel tanımlanır:

1. **Subaccount Tanımlama:** SCC konsolundan BTP Subaccount (Region, Subaccount ID, Subaccount User) eklenir.
2. **S/4HANA Sistem Eşlemesi (Cloud To On-Premise):**
   * *Backend Type:* ABAP System.
   * *Protocol:* HTTPS.
   * *Internal Host:* `s4hana.internal.corp:44300` $\rightarrow$ *Virtual Host:* `s4hana-scim.corp.dummy:443`.
3. **Erişilebilir Kaynaklar (Accessible Resources):**
   * URL Path: `/sap/bc/scim` (Path and all sub-paths).
   * RFC/BAPI kullanılacaksa: BAPI fonksiyon modülü prefixleri (`BAPI_USER_*`).

#### 1.3. S/4HANA Tarafındaki Servis ve Sertifika Aktivasyonları

1. **ICF Servis Aktivasyonu (`SICF`):** `/default_host/sap/bc/scim` ağacı aktif hale getirilir.
2. **Sertifika ve SSL Yapılandırması (`STRUST`):** SCC'nin sanal anahtar sertifikası S/4HANA `SSL Server Standard` PSE içine güvenilen sertifika (Trust Certificate) olarak eklenir.
3. **Teknik Kullanıcı:** IPS'in SCIM çağrılarını karşılayacak en düşük yetkili bir sistem kullanıcısı (`SVC_IPS_SCIM`) tanımlanır ve `SAP_BC_SCIM_ADMIN` yetki rolü atanır.

> [!NOTE]
> **Operasyonel Not (Lisans / FUE):**  
> S/4HANA tarafında kullanıcı açılırken varsayılan lisans tipinin kontrolsüz atanmasını önlemek adına, `SU01` üzerinde varsayılan sözleşmesel kullanıcı tipi (*Professional*, *Functional* veya *Productivity*) modül yetki ekibiyle önceden kararlaştırılmalıdır.

#### 1.4. Mevcut (Brownfield) S/4HANA Kullanıcılarının Ön Eşitlenmesi (Pre-Migration Data Cleanup)

Canlıya geçiş öncesinde SAP Basis ekibi mevcut aktif `SU01` kullanıcılarının `externalId` / `Alias` alanlarını İK personel numarasıyla (`employeeID`) eşitlemelidir (`SU10` veya toplu ABAP güncelleme raporu ile).

* **Neden Zorunlu?** Mevcut kullanıcıların `externalId` alanı boş bırakılırsa, IPS ilk çalıştığında bu kullanıcıları SCIM üzerinden eşleştiremez. Aynı `userName` ile yeni bir kullanıcı yaratmaya çalışır ve sistem `HTTP 409 Conflict` (kullanıcı zaten mevcut) hatası üreterek senkronizasyonu durdurur.
* **Operasyonel Çıktı:** Tüm aktif personelin `SU01 Alias` = `employeeID` eşleşmesi tamamlanmadan Faz 3'teki hedef sistem aktif edilmemelidir.

---

### FAZ 2: Kimlik Federasyonu ve Güven (Trust) Kurulumu

Bu fazda kullanıcıların giriş yaparken şifre girecekleri kurumsal SSO hattı oluşturulur:

```
[Kullanıcı Tarayıcısı] ──► [BTP App / Launchpad] ──► [SAP IAS (Proxy)] ──► [Entra ID (MFA / IdP)]
```

#### 2.1. Entra ID $\leftrightarrow$ SAP IAS Güven İlişkisi (Corporate Identity Provider)

1. **IAS Konsoluna Giriş:** `https://<tenant-id>.accounts.ondemand.com/admin/` adresine gidilir.
2. **Corporate IdP Ekleme:** *Identity Providers* $\rightarrow$ *Corporate Identity Providers* $\rightarrow$ *Create* adımı seçilir:
   * Protokol olarak **OpenID Connect (OIDC)** veya **SAML 2.0** seçilir.
   * Entra ID Federasyon Metadata XML'i (veya OIDC Discovery URL'i) yüklenir.
3. **Identity Federation ve Claim Ayarları:**
   * *Unique User Attribute:* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` $\rightarrow$ `employeeID`.
   * *Group Claim:* Entra ID'nin gönderdiği `groups` token talebi IAS üzerinde harici grup niteliği olarak karşılanır.
4. **Tek Merkezli MFA Ayarı (Single-IdP MFA):** IAS üzerindeki *Risk-Based Authentication* kuralları kapatılarak tüm koşullu erişim ve MFA sorumluluğu kurumsal Entra ID tarafına devredilir (Pass-Through).

#### 2.2. SAP IAS $\leftrightarrow$ BTP Subaccount Güven İlişkisi

1. **BTP Cockpit'e Giriş:** Subaccount $\rightarrow$ *Security* $\rightarrow$ *Trust Configuration* sekmesine gidilir.
2. **Establish Trust:** *Establish Trust* butonuna basılarak bağlı SAP Cloud Identity Services tenant'ı otomatik eşleştirilir (OIDC protokolüyle).
3. **Assertion Attributes Tanımlama:**
   * Attribute `Groups` $\rightarrow$ IAS'tan gelen `groups` claim'ine eşlenir.
   * Attribute `CompanyCode` (ABAC kullanılacaksa) $\rightarrow$ Kullanıcının şirket kodu alanına bağlanır.

---

### FAZ 3: IPS Veri Dağıtım Motorunun Kurulumu ve Hedef Sistemler

Kullanıcıların ve grupların arka planda dağıtılacağı asenkron SCIM boru hattı kurulur:

#### 3.1. Kaynak Sistem (Source System) Kurulumu: Entra ID

*IPS Konsolu* $\rightarrow$ *Source Systems* $\rightarrow$ *Create*:
* **System Type:** `Microsoft Entra ID`
* **Properties:**
  * `Type: HTTP`
  * `Authentication: BasicAuthentication / OAuth2ClientCredentials`
  * `entra.id.client.id: <Faz 1'den alınan Client ID>`
  * `entra.id.client.secret: <Faz 1'den alınan Secret>`
  * `entra.id.tenant.id: <Faz 1'den alınan Tenant ID>`
  * `entra.id.filter.groups: displayName startswith 'SG_SAP_'` (Network trafiğini optimize etmek için sunucu taraflı filtre).

#### 3.2. Hedef Sistem (Target System 1) Kurulumu: SAP S/4HANA On-Premise / PCE

*IPS Konsolu* $\rightarrow$ *Target Systems* $\rightarrow$ *Create*:
* **System Type:** `SAP S/4HANA On-Premise`
* **Destination (BTP Cockpit Destinasyonları Üzerinde Tanımlanır):**
  * `URL: http://s4hana-scim.corp.dummy:443/sap/bc/scim`
  * `ProxyType: OnPremise` (Cloud Connector üzerinden geçiş)
  * `Authentication: BasicAuthentication` (Teknik kullanıcı: `SVC_IPS_SCIM`)
* **Properties (Emniyet Parametreleri):**
  * `ips.delete.threshold = 10`
  * `s4hana.authoritative.role.management = false` (İlk devreye alma fazında korunmalı)
  * `ips.target.options.role.exclude.filter = (Z_FF_.*|Z_EXC_.*)`
  * `ips.trace.failed.entity.content = true`
  * `ips.job.bulk.page.size = 100`

#### 3.3. Hedef Sistem (Target System 2) Kurulumu: SAP Cloud Identity Services (Identity Directory)

* **System Type:** `SAP Cloud Identity Services`
* **Properties:**
  * `Type: HTTP`
  * `URL: https://<tenant-id>.accounts.ondemand.com/service/scim`
  * `Authentication: ClientCertificateAuthentication` (IAS üzerinden üretilen teknik API sertifikası).

---

### FAZ 4: Çalışma Zamanı Yetki Eşleme (Role Mapping) Yapılandırması

Modüller kendi mutfaklarında rolleri tasarladıktan sonra bu rollerin kurumsal gruplarla fiziki bağı kurulur:

```
[Active Directory Grubu]                 [Eşleştirme Katmanı]               [Hedef Sistem Rolü]
SG_SAP_S4H_FI_GL_1000     ──► (IPS JSON Mapping)             ──► S/4HANA: Z_BR_GL_ACCOUNTANT_1000
SG_SAP_BTP_DEV_MGR        ──► (BTP Cockpit Role Collection)  ──► BTP: Procurement_Manager_Collection
```

#### 4.1. BTP Subaccount Seviyesinde Role Collection Eşleştirmesi

BTP uygulamaları için kullanıcı kopyalaması yapılmaz; anlık eşleme kuralı girilir:

1. **BTP Cockpit** $\rightarrow$ **Security** $\rightarrow$ **Role Collections**.
2. Hedef rol koleksiyonuna tıklanır (Örn: `Procurement_Manager_Collection`).
3. **User Groups** tabında **Add** seçeneğiyle:
   * *Identity Provider:* `SAP Cloud Identity Services`
   * *Name:* `SG_SAP_BTP_PROCUREMENT_MGR` (AD grubuyla harfi harfine aynı).

#### 4.2. S/4HANA Tarafında Principal Propagation (X.509 Kural Tanımlama)

BTP uygulamasından S/4HANA'ya gelen çağrılarda kullanıcının kimliğinin korunması için:

1. **İşlem Kodu `CERTRULE`:** S/4HANA üzerinde sertifika eşleme kuralı açılır:
   * *Issuer DN:* SCC kurumsal CA sertifikasının DN bilgisi.
   * *Subject Component:* `CN` (Common Name) $\rightarrow$ `SU01 Alias` veya `User ID` eşleşmesi kuralı tanımlanır.
2. Bu sayede SCC'den gelen dinamik X.509 sertifikası doğrudan doğruya S/4HANA'daki yerel kullanıcı hesabına bağlanır.

---

### FAZ 5: Sistemin Oturması, Test ve Doğrulama Adımları

Sistemin canlıya güvenle oturması için devreye alma süreci şu sıralı adımlarla işletilir:

| Test / Doğrulama Adımı | Yapılacak İşlem | Beklenen Çıktı / Doğrulama |
|:---|:---|:---|
| **1. Simülasyon Çalıştırması (IPS Simulation Mode)** | IPS konsolunda hedef sistem seçilir ve *Run Simulation* tetiklenir. | Gerçek sisteme yazma yapılmaz; üretilen CSV raporunda eklenecek, güncellenecek kullanıcı ve rol sayıları incelenir. |
| **2. Tekil Pilot Senkronizasyonu** | Entra ID tarafında yalnızca bir pilot kullanıcı `SG_SAP_S4H_FI_GL_1000` grubuna eklenir. | IPS delta sync çalıştırılır; S/4HANA `SU01` üzerinde kullanıcının oluştuğu ve `Z_BR_GL_ACCOUNTANT_1000` rolünü aldığı doğrulanır. |
| **3. Leaver Kilit Testi** | Pilot kullanıcının Entra ID hesabı devre dışı bırakılır (`accountEnabled = false`). | IPS senkronizasyonu sonrası S/4HANA `SU01` hesabının kilitlendiği (`Lock`), BTP oturumunun kapandığı teyit edilir. |
| **4. Eşik Değeri (Threshold) Doğrulaması** | Test ortamında toplu grup çıkarma simüle edilir. | `ips.delete.threshold = 10` kuralının devreye girerek senkronizasyon job'ını acil durdurduğu ve log bastığı doğrulanır. |
| **5. Firefighter Koruma Testi** | S/4HANA'da bir kullanıcıya elle `Z_FF_FINANCE` rolü verilir. | IPS sync tetiklenir; `s4hana.authoritative.role.management = false` ve regex filtreleri sayesinde elle verilen rolün silinmediği görülür. |

> [!TIP]
> **Yetki Kayması Notu (Continuous Drift):**  
> Sistem oturduktan ve kararlı hale geldikten sonra, ayda bir kez IPS dışından S/4HANA içine elle atanmış yetkilerin denetim raporu alınmalı; temizlenen kayıtların ardından kademeli olarak `s4hana.authoritative.role.management = true` moduna geçilmelidir.

---

## 8. KRİTİK GÜVENLİK, DENETİM VE OPERASYONEL IPS PARAMETRELERİ

Hedef sistem seviyesinde tanımlanan operasyonel güvenlik parametreleri referans tablosu:

| Parametre / Konfigürasyon | Standart Değer | Teknik Fonksiyonu & Operasyonel Referansı |
|:---|:---|:---|
| **`ips.delete.threshold`** | `10` | **Toplu İşlem Eşiği:** Tek job döngüsünde kullanıcıların %10'undan fazlasının silinmesini/kilitlenmesini engelleyerek işlemi acil durdurur (Bkz. Giriş & Faz 5.4). |
| **`s4hana.authoritative.role.management`** | `false` *(Geçiş)*<br>`true` *(Kararlı)* | **Yetki Ezme Kontrolü:** Geçiş fazında yerel manuel rolleri korur (`false`); temizlik sonrası kararlı fazda merkezi otoriter moda alınır (`true`). |
| **`ips.trace.failed.entity.content`** | `true` | **Hata Teşhis (Troubleshooting):** Hatalı SCIM kayıtlarının payload içeriğini KVKK maskelemesiyle loglara yazar. |
| **`ips.job.bulk.page.size`** | `100` | **Performans & Rate-Limit:** SCIM REST çağrılarında network yükünü ve bellek tüketimini sınırlar. |
| **`ips.target.options.role.exclude.filter`** | `(Z_FF_.*\|Z_EXC_.*)` | **İstisna Rol Koruması:** Firefighter, acil müdahale ve istisnai teknik rollerin IPS senkronizasyonunda silinmesini engeller. |
| **`entra.id.filter.groups`** | `displayName startswith 'SG_SAP_'` | **Graph API Throttling Koruması:** Entra ID üzerindeki binlerce grubun çekilerek Graph API paket sınırlarını aşmasını ve IPS timeout hatalarını engeller. |
| **`SU01 Alias = employeeID`** | Pre-Migration Şartı | **Brownfield Çakışma Önleme:** Mevcut S/4HANA kullanıcılarının `externalId` alanına personel numarası yazılarak SCIM 409 Conflict hataları önlenir. |
| **MFA Mimari Modeli** | Single-IdP (Pass-Through) | **Çift MFA Önleme:** IAS Risk-Based MFA kapatılır; tüm doğrulama Entra ID Conditional Access'e devredilir (Bkz. Faz 2.1). |

---

## 9. KURUMSAL KİMLİK OLGUNLUK MODELİ VE SIFIR GÜVEN (ZERO TRUST - NIST SP 800-207) MİMARİSİ

### A. IAM Kurumsal Olgunluk Modeli (IAM Maturity Model)
Kurumların kimlik ve erişim yönetimi süreçleri endüstri standardı 4 olgunluk seviyesinde sınıflandırılır:

```
[ Seviye 1: Kaotik / Manuel ] ──► [ Seviye 2: Parçalı / IT Odaklı ] ──► [ Seviye 3: Kural Tabanlı JML ] ──► [ Seviye 4: Sıfır Güven & AI ]
- IT'ye açılan biletler          - AD scriptleri var                 - SuccessFactors tetikler            - Risk tabanlı adaptif erişim
- Excel'de tutulan yetkiler      - SAP sistemleri bağımsız           - Sıfır manuel bilet (Zero-Touch)     - Otomatik SoD denetimi (IAG)
- Haftalar süren onaylar         - Yetki birikimi ve kayması yüksek  - Kural tabanlı otomatik provizyon   - Anomali ve bağlam analizi
```

* **Seviye 1 (Kaotik / Manuel - Reaktif):** İşe girişlerde İK, IT'ye e-posta/bilet açar. SAP Basis uzmanı `SU01` ekranında manuel kullanıcı ve rol tanımlar. İşten ayrılanların yetkileri haftalarca kapatılamaz; yetim hesap (orphan account) oranı %10'u aşar.
* **Seviye 2 (Parçalı / Silo Yapı):** Active Directory tarafında bazı otomatikleştirilmiş scriptler bulunur; ancak SAP ekosistemi (S/4HANA, BTP, SAC) bu otomasyonun dışındadır. Senkronizasyon manuel biletlerle sağlanır.
* **Seviye 3 (Kural Tabanlı Otomasyon / JML Standardı):** İK ana veri kaynağındaki (SAP SuccessFactors) sözleşme veya pozisyon değişikliği tek güven kaynağıdır (Single Source of Truth). Entra ID, SAP IAS, BTP ve S/4HANA üzerinde roller **Zero-Touch** olarak otomatik açılır, güncellenir ve dondurulur.
* **Seviye 4 (Sıfır Güven & AI Destekli - İleri Düzey):** Kimlikler yalnızca statik unvana göre değil; kullanıcının anlık bağlamına (cihaz durumu, IP, anomali skoru) ve SAP Cloud Identity Access Governance (IAG) ile gerçek zamanlı Görevler Ayrılığı (SoD) kontrollerine göre dinamik olarak yönetilir.

### B. NIST SP 800-207 Sıfır Güven (Zero Trust) Hizalanması
Bu mimari model, ABD Ulusal Standartlar ve Teknoloji Enstitüsü'nün (NIST) **SP 800-207 Zero Trust Architecture** standardında tanımlanan 3 çekirdek ilkeyi eksiksiz uygular:

1. **Ağa Değil, Kimliğe Güven (Explicit Verification):**
   * Kullanıcının şirket ağında veya kurumsal VPN'de olması ona doğrudan erişim hakkı tanımaz. Her işlemde Entra ID $\rightarrow$ IAS $\rightarrow$ BTP XSUAA zincirinde kimlik, token geçerliliği ve güvenlik grubu üyelikleri açıkça doğrulanır.
2. **En Az Yetki İlkesi (Principle of Least Privilege):**
   * Kullanıcılara geniş/jenerik roller yerine, yalnızca SF üzerindeki doğrulanmış görev tanımına (`JobCode`) ve türetilmiş organizasyonel kısıtlarına (`CompanyCode`) karşılık gelen atomik yetkiler verilir.
3. **Kısa Ömürlü İtimat (Ephemeral Trust & Principal Propagation):**
   * Sistemler arasında statik teknik kullanıcı şifreleri (Hardcoded Service Passwords) paylaşılmaz. BTP'den arka uç S/4HANA sistemine erişirken SAP Cloud Connector (SCC) tarafından dinamik olarak üretilen **2 dakikalık geçici X.509 istemci sertifikaları** kullanılır. Çalınabilecek statik bir şifre bulunmaz ve denetim izinde işlemi yapan gerçek kullanıcının kimliği (Principal) görünür.

---

## 10. İSTİSNAİ ERİŞİM YÖNETİMİ: GÖREVLER AYRILIĞI (SoD) VE ONAY MEKANİZMALARI

Bölüm 5.4'teki taktiksel sınır kılavuzunda belirtildiği üzere; kimlik yaşam döngüsü otomasyonu, çalışanların sözleşmelerine ve görevlerine bağlı standart doğuştan gelen (Birthright) yetkileri (%80 hacim) hedefler. Geriye kalan %20'lik istisnai, finansal limit içeren veya geçici yetkiler ise denetim riskini sıfırlamak adına **Görevler Ayrılığı (Segregation of Duties - SoD)** ve **Çift Onay İş Akışları** çerçevesinde yönetilir:

### 10.1. Görevler Ayrılığı (SoD) ve Kritik Çıkar Çatışmaları
* **SoD Temel Prensibi:** Tek bir personelin finansal kayıp, hile veya suistimale yol açabilecek birbirini denetleyen iki kritik işlem adımını aynı anda yürütmesi sistem seviyesinde engellenir.
* **Klasik SAP Çakışma Matrisi Örnekleri:**
  * *Satıcı Ana Verisi Yönetimi (`FK01` / `XK01`)* $\leftrightarrow$ *Fatura Girişi ve Ödeme Onayı (`F110` / `FB60`)*.
  * *Satın Alma Siparişi Verme (`ME21N`)* $\leftrightarrow$ *Mal Kabul / Giriş Onayı (`MIGO`)*.
  * *Müşteri Kredi Limiti Tanımlama (`FD32`)* $\leftrightarrow$ *Satış Siparişi Onaylama (`VA01`)*.
* **Otomasyon Sınır Kuralı:** IPS motoru, kullanıcının pozisyonu gerektirse dahi bu tip yüksek riskli SoD çakışmalarını **aynı anda otomatik olarak tek kullanıcıya bağlayamaz**. Standart yetki otomatik verilirken, çatışan kritik ikinci yetki mutlaka onaylı talep akışına bırakılır.

### 10.2. Talep Tabanlı Yetki Yaşam Döngüsü ve Onay Katmanı
* **Çift Onay İlkesi (Four-Eyes Principle):** Yüksek limitli finansal onaylar veya sistem parametre rolleri için talep, doğrudan çalışanın amiri ve ilgili iş süreci sahibinin (Business Process Owner) ortak dijital onayıyla açılır.
* **SAP IAG / GRC Entegrasyon Köprüsü:** İleri fazda kurulacak olan **SAP Cloud Identity Access Governance (IAG)** veya SAP GRC Access Control çözümü; talep anında simülasyon çalıştırarak kullanıcının mevcut yetkileriyle yeni talep arasında SoD riski olup olmadığını gerçek zamanlı denetler; risksiz ise geçici süreliğine aktif eder.
* **Periyodik Yetki Gözden Geçirme (Access Recertification):** Yılda en az bir kez birim yöneticilerine açık olan tüm istisnai yetkilerin listesi otomatik onay formlarıyla iletilir; onaylanmayan geçici yetkiler süre sonunda otomatik sonlandırılır.

---

## 11. OPERASYONEL METRİKLER VE TEKNİK KATMA DEĞER ANALİZİ

Kimlik yaşam döngüsü otomasyonunun kurumsal operasyonlara ve sistem güvenliğine sağladığı ölçülebilir kazanımlar:

| Metrik / KPI | Manuel Süreç (Mevcut Durum) | Otomasyon Sonrası (Hedeflenen) | Sağlanan Teknik & Operasyonel Değer |
|:---|:---|:---|:---|
| **Erişim Hazırlık Süresi (Time-to-Productivity)** | **3 ila 7 iş günü** (Bilet onayları, SAP Basis bekleme süreleri). | **< 1 Saat (Day-1 Ready)** | Yeni çalışan ilk mesai gününde tüm SAP ekranlarına erişerek anında katma değer üretmeye başlar. |
| **IT Destek Yükü (Ticket Volume)** | Ayda yüzlerce "SAP yetki talebi", "şifre sıfırlama", "hesap açılışı" bileti. | **%80 - %90 Azalma** | IT ve SAP Basis ekipleri operasyonel bilet çözmek yerine stratejik projelere odaklanır. |
| **Yetim Hesap (Orphan Account) Oranı** | İşten ayrılanların ortalama %5-10'unun SAP hesapları haftalarca açık kalır. | **%0 (Sıfır Tolerans)** | Sözleşme feshedildiği saniye tüm oturumlar kilitlenir; eski çalışan veri sızıntısı riski son bulur. |
| **Denetim Hazırlık Süresi (Audit Readiness)** | Yılda 2 kez yapılan denetimlerde ekran görüntüleri ve Excel logları toplamak haftalar sürer. | **Dakikalar İçinde Tek Tıkla** | Denetçilere IPS senkronizasyon logları ve IAS Audit Trail doğrudan resmi delil olarak sunulur. |

---

## 12. OLAĞANÜSTÜ DURUM MÜDAHALE (BREAK-GLASS), FELAKET KURTARMA VE KVKK/GDPR UYUM ÇERÇEVESİ

### A. Acil Durum Yönetici Hesapları (Break-Glass Protocol)
* **Senaryo:** Microsoft Entra ID veya SAP Cloud Identity Services genelinde küresel bir bulut kesintisi (Cloud Outage) yaşanması veya federasyon sertifikasının süresinin dolması.
* **Müdahale Prosedürü (Break-Glass):**
  1. SAP S/4HANA ve SAP BTP subaccount seviyelerinde federasyondan bağımsız, doğrudan yerel veritabanında tanımlı **acil durum süper yönetici hesapları** (`BREAK_GLASS_ADMIN`) barındırılır.
  2. Bu hesapların şifreleri insan belleğinde tutulmaz; kurumsal ayrıcalıklı erişim kasasında (**CyberArk Enterprise Password Vault** veya **Azure Key Vault HSM**) kilitli tutulur.
  3. Kasa şifresi yalnızca çift onay (Dual-Control / Four-Eyes Principle) ile açılabilir. Kasadan şifre çekildiği saniye SOC (Security Operations Center) ekibine en yüksek seviye (P1) güvenlik alarmı üretilir.

### B. Simülasyon Modu (Simulation Mode) ve Geri Alma (Rollback) Güvencesi
Yeni bir yetki kuralı veya IPS JSON dönüşümü canlı ortama alınmadan önce kesinlikle doğrudan çalıştırılmaz:
1. **Simulation Job:** IPS üzerinde hedef sistem için **"Run Simulation"** seçeneği tetiklenir.
2. **Etki Analizi Raporu:** Simülasyon sonucunda üretilen CSV dosyası incelenir (güncellenecek, kilitlenecek ve rol eklenecek/silinecek kullanıcı sayıları doğrulanır).
3. **Rollback Stratejisi:** Beklenmeyen bir toplu değişiklik tespit edilirse, JSON transformation versiyon geçmişinden (Version History) tek tıkla önceki stabil sürüme dönülür.

### C. Regülasyon ve Veri Gizliliği Uyumu (KVKK / GDPR / ISO 27001)
* **Veri Asgarileştirmesi (Data Minimization - GDPR Md. 5(1)(c) / KVKK Md. 4):** SAP IPS, Active Directory'den kullanıcının ev adresini, özel telefonunu veya kişisel notlarını kesinlikle çekmez. Sadece kimlik doğrulama ve yetkilendirme için zorunlu olan 5 temel alan taşınır (`userName`, `email`, `employeeId`, `department`, `groups`).
* **Değişmez Denetim İzi (Immutable Audit Trail - ISO/IEC 27001:2022 Kontrol 8.15):** Kimin hangi gruba ne zaman girdiği, yetkinin ne zaman verildiği ve hangi tarihte sistemden alındığı bilgisi SAP IAS Audit Log servisinde kriptografik olarak imzalanmış şekilde saklanır; KVKK, BDDK ve ISO 27001 denetimlerine doğrudan resmi delil teşkil eder.

---

## 13. KRİTİK REGÜLASYON VE UYUMLULUK TAKVİMİ

> [!IMPORTANT]
> **SAP SuccessFactors Temel Şifreli Girişlerin Kaldırılması (Deprecation of Basic Authentication):**  
> SAP resmi direktifi ve **SAP KBA 2791410** uyarınca, **13 Kasım 2026** tarihi itibarıyla SuccessFactors için temel şifre (Basic Authentication) desteği kalıcı olarak sonlandırılacaktır. Bu tarihten sonra tüm SF kullanıcıları ve API entegrasyonları için **SAP Cloud Identity Services - Identity Authentication (IAS)** kullanımı zorunludur.  
> *Resmi Kaynak: [SAP KBA 2791410 - Deprecation of Basic Authentication in SAP SuccessFactors](https://me.sap.com/notes/2791410)*

---

## 14. KAPSAMLI TERİMLER VE KISALTMALAR SÖZLÜĞÜ (GLOSSARY & ACRONYMS)

Bu bölüm, SAP IAM/JML kurumsal mimari tasarımında ve sahadaki teknik paydaş toplantılarında kullanılan terim, kavram ve kısaltmaların resmi referans sözlüğüdür.

### 14.1. Kısaltmalar Sözlüğü (Acronyms & Abbreviations)

| Kısaltma | Açılımı (İngilizce / Orijinal) | Türkçe Anlamı & Mimarideki Teknik Fonksiyonu |
|:---|:---|:---|
| **ABAC** | Attribute-Based Access Control | Nitelik Tabanlı Erişim Denetimi. Kullanıcının unvan, departman, şirket kodu gibi anlık profil niteliklerine göre veri kısıtlaması (BTP CAP CDS katmanı). |
| **API** | Application Programming Interface | Uygulama Programlama Arayüzü. Sistemlerin birbiriyle veri alışverişi yaptığı programatik bağlantı noktası. |
| **BAPI** | Business Application Programming Interface | SAP On-Premise sistemlerinde standart iş nesnelerine dışarıdan erişim sağlayan RFC tabanlı fonksiyon modülü (Örn: `BAPI_USER_CREATE1`). |
| **BP** | Business Partner | İş Ortağı. S/4HANA üzerinde müşteri, satıcı ve çalışanların konsolide edildiği merkezi ana veri varlığı. |
| **BTP** | SAP Business Technology Platform | SAP'nin bulut entegrasyon, geliştirme ve kimlik doğrulama omurgası. |
| **CA** | Certificate Authority | Sertifika Otoritesi. X.509 dijital sertifikalarını imzalayan güvenilir kurumsal veya küresel kök makam. |
| **CAP** | Cloud Application Programming Model | SAP BTP üzerinde Java ve Node.js ile kurumsal bulut uygulamaları geliştirmek için kullanılan resmi mimari çerçeve. |
| **CDS** | Core Data Services | SAP HANA ve CAP üzerinde veri modellerini ve deklaratif erişim kurallarını (`@restrict`) tanımlayan şema dili. |
| **CVI** | Customer Vendor Integration | Müşteri-Satıcı-Çalışan Entegrasyonu. S/4HANA'da HCM personel kaydı ile İş Ortağı (BP) ve `SU01` kullanıcısını birbirine bağlayan zorunlu senkronizasyon motoru. |
| **DN** | Distinguished Name | X.509 ve LDAP dizin yapılarında bir sertifika veya nesnenin benzersiz tam hiyerarşik adı (Örn: `CN=s4hana, OU=IT, O=Corp, C=TR`). |
| **FUE** | Full User Equivalent | Tam Kullanıcı Eşdeğeri. SAP bulut lisanslamasında farklı kullanıcı tiplerini (Professional = 1.0, Functional = 0.25, Productivity = 0.10) tek bir katsayı havuzunda toplayan sözleşmesel metrik birim. |
| **GRC** | Governance, Risk, and Compliance | Kurumsal Yönetişim, Risk ve Uyumluluk. Şirket içi yetki risklerini ve yasal denetimleri yöneten SAP yazılım ailesi. |
| **HSM** | Hardware Security Module | Donanımsal Güvenlik Modülü. Kriptografik anahtarları ve kök sertifikaları fiziksel olarak koruyan yüksek güvenlikli donanım kasası. |
| **IAG** | SAP Cloud Identity Access Governance | SAP BTP üzerinde çalışan, bulut ve on-premise sistemlerde gerçek zamanlı Görevler Ayrılığı (SoD) denetimi ve yetki talep iş akışı sağlayan SaaS çözümü. |
| **IAM** | Identity and Access Management | Kimlik ve Erişim Yönetimi. Doğru kişilerin doğru kaynaklara doğru zamanda ve doğru nedenlerle erişmesini sağlayan kurumsal disiplin. |
| **IAS** | SAP Cloud Identity Services - Identity Authentication | SAP bulut ekosisteminin kurumsal kimlik doğrulama, SSO ve OIDC/SAML federasyon proxy hub servisi. |
| **ICF** | Internet Communication Framework | SAP ABAP uygulama sunucusunun HTTP/HTTPS web isteklerini karşıladığı çekirdek servis ağacı (İşlem kodu: `SICF`). |
| **IdP** | Identity Provider | Kimlik Sağlayıcı. Kullanıcıların kimliğini doğrulayan ve diğer sistemlere doğrulanmış token (bilet) üreten güven kaynağı (Örn: Microsoft Entra ID). |
| **IPS** | SAP Cloud Identity Services - Identity Provisioning | Kullanıcı ve grupları kaynak dizinlerden (Entra ID, SF) okuyup hedef sistemlere (S/4HANA, BTP, IAS) SCIM protokolüyle dağıtan asenkron ETL motoru. |
| **JIT** | Just-In-Time Provisioning | Tam Zamanında Provizyon. Kullanıcı hesabının önceden hedef sisteme açılmayıp, ilk başarılı oturum açma (login) anında token taleplerine göre dinamik oluşturulması veya anlık rol koleksiyonuna eşlenmesi. |
| **JML** | Joiner, Mover, Leaver | İşe Giriş (Joiner), Görev Değişimi (Mover) ve İşten Ayrılış (Leaver) adımlarını kapsayan uçtan uca çalışan kimlik yaşam döngüsü. |
| **MFA** | Multi-Factor Authentication | Çok Faktörlü Doğrulama. Kullanıcının oturum açarken parola haricinde ek bir doğrulama unsuru (SMS, Authenticator App, FIDO2) girmesini zorunlu kılan güvenlik katmanı. |
| **NIST** | National Institute of Standards and Technology | ABD Ulusal Standartlar ve Teknoloji Enstitüsü. Sıfır Güven (Zero Trust - SP 800-207) mimarisinin küresel referans otoritesi. |
| **OIDC** | OpenID Connect | OAuth 2.0 protokolü üzerine inşa edilmiş, JSON Web Token (JWT) tabanlı modern kimlik doğrulama standardı. |
| **PFCG** | Profile Generator | S/4HANA ABAP sisteminde yetki rollerinin, işlem kodlarının ve yetki nesnelerinin tasarlandığı temel yönetim işlem kodu. |
| **PIM** | Privileged Identity Management | Ayrıcalıklı Kimlik Yönetimi. Süper yönetici yetkilerinin sürekli açık kalması yerine ihtiyaç anında süreli olarak onay ile verilmesi (Örn: Entra ID PIM). |
| **PKI** | Public Key Infrastructure | Açık Anahtar Altyapısı. Sayısal sertifikaların üretimi, dağıtımı ve doğrulanmasını sağlayan kurumsal şifreleme omurgası. |
| **PSE** | Personal Security Environment | SAP ABAP sisteminde SSL/TLS istemci ve sunucu sertifikalarını, güven zincirlerini saklayan yerel sertifika kasası (İşlem kodu: `STRUST`). |
| **RBAC** | Role-Based Access Control | Rol Tabanlı Erişim Denetimi. Yetkilerin doğrudan kullanıcılara değil, iş fonksiyonlarını temsil eden rollere atanması prensibi. |
| **RFC** | Request for Comments / Remote Function Call | 1. IETF internet standart belgesi (Örn: RFC 7643).<br>2. SAP sistemleri arasında veri ve fonksiyon çağrısı sağlayan ikili iletişim protokolü. |
| **SAML** | Security Assertion Markup Language | XML tabanlı, kurumlar arası tekil oturum açma (SSO) ve yetki beyanı sağlayan olgun federasyon protokolü (SAML 2.0). |
| **SCC** | SAP Cloud Connector | SAP Bulut Konnektörü. BTP ile şirket içi (On-Premise / PCE) güvenli ağ arasında dışarıdan port açmadan ters tünel (reverse tunnel) kuran DMZ yazılımı. |
| **SCIM** | System for Cross-domain Identity Management | IETF RFC 7643/7644 standartlarına dayalı, REST/JSON tabanlı bulut kimlik provizyonu ve şema protokolü. |
| **SF** | SAP SuccessFactors | İK ana verisinin, pozisyonların ve istihdam sözleşmelerinin tutulduğu tek güven kaynağı (Single Source of Truth) bulut HCM sistemi. |
| **SoD** | Segregation of Duties | Görevler Ayrılığı. Finansal veya operasyonel suistimalleri önlemek amacıyla birbirini kontrol eden kritik işlemlerin farklı kişilere verilmesi kuralı. |
| **SSO** | Single Sign-On | Tekil Oturum Açma. Kullanıcının tek bir kullanıcı adı ve şifreyle kurumun tüm entegre uygulamalarına oturum açabilmesi kabiliyeti. |
| **UPN** | User Principal Name | Active Directory ve Entra ID'de kullanıcının e-posta biçimindeki oturum açma adı (Örn: `ahmet.yilmaz@corp.com`). |
| **XSUAA** | Extended Services User Account and Authentication | SAP BTP Cloud Foundry ortamında OAuth 2.0 token üreten, Role Collection ve scope atamalarını yöneten yetkilendirme çekirdeği. |

---

### 14.2. İçerik ve Kavramsal Terimler Sözlüğü (Glossary of Core Concepts)

* **Authoritative Mode (Otoriter Senkronizasyon):** SAP IPS üzerinde bir hedef sistem için bu parametre (`authoritative = true`) açıldığında; sistem, merkezi kaynakta (AD) bulunmayan ancak hedef sistemde (S/4HANA) elle atanmış tüm yerel yetkileri kalıcı olarak siler. Geçiş aşamasında operasyonel işlerin aksamaması için `false` tutulmalıdır.
* **Birthright Access (Doğuştan Gelen Yetki):** Çalışanın unvanı, pozisyon kodu (`JobCode`) ve departmanı gereği işe başladığı ilk gün (Day-1) herhangi bir ek onaya ihtiyaç duymaksızın otomatik olarak atanması gereken standart, risksiz erişim paketi (Fiori Launchpad, ESS izin, bordro, kurumsal BTP portalı).
* **Break-Glass Protokolü (Acil Durum Kasası):** Bulut sağlayıcılarında küresel kesinti yaşanması veya federasyon sertifikalarının düşmesi anında; iş sürekliliğini sağlamak amacıyla yerel süper yönetici hesaplarının şifrelerinin kurumsal dijital kasadan (CyberArk/HSM) çift onay ile çekilerek kullanılması prosedürü.
* **Brownfield SU01 Eşitlemesi (Pre-Migration Data Alignment):** Mevcut S/4HANA sistemlerinde önceden elle açılmış kullanıcıların `SU01 Alias` veya `externalId` alanlarının İK personel numarasıyla (`employeeID`) canlıya geçiş öncesinde birebir eşitlenmesi işlemi. Yapılmadığında IPS kullanıcının varlığını algılayamaz ve mükerrer kayıt açmaya çalışırken SCIM 409 Conflict çakışması üretir.
* **Continuous Drift (Sürekli Yetki Kayması):** Merkezi otomasyon boru hattı (IPS) haricinde, yetki yöneticileri tarafından S/4HANA `SU01` ekranında doğrudan elle rol atanması sonucunda merkezi dizin ile yerel sistem arasında oluşan uyumsuzluk ve denetim sapması.
* **Graph API Throttling (Hız Sınırlaması):** Microsoft Entra ID üzerinde çok sayıda grup veya iç içe nesne filtresiz çekilmeye çalışıldığında, Microsoft Graph API servislerinin aşırı yükü engellemek amacıyla HTTP 429 Too Many Requests yanıtı döndürerek senkronizasyonu durdurması veya yavaşlatması durumu. `displayName startswith 'SG_SAP_'` filtresiyle aşılır.
* **Corporate Identity Provider (Kurumsal Kimlik Sağlayıcı):** Bir şirketin tüm kullanıcı veritabanını, şifre politikalarını ve MFA kurallarını merkezi olarak barındıran tekil güven kaynağı (Örn: Microsoft Entra ID). SAP IAS bu yapıda kurumsal IdP'nin önüne proxy olarak konumlanır.
* **Değişmez Anahtar (Immutable Correlation Key):** Kullanıcının adı, soyadı, kurumsal e-postası veya unvanı değişse dahi hiçbir koşulda değişmeyen tekil anahtar (`personIdExternal` / `employeeID` / `userUuid`). Tüm sistemler arası senkronizasyon eşleşmesi bu anahtara bağlanmalıdır.
* **Derived Roles (Türetilmiş Roller):** S/4HANA PFCG mimarisinde; menü yapısı ve işlem kodları bir üst ana rolden (Parent Role) miras alınan, ancak şirket kodu, masraf merkezi veya tesis gibi organizasyonel seviyeleri spesifik olarak doldurulmuş alt yetki rolleri (Örn: `Z_BR_GL_ACCOUNTANT_1000`).
* **Double-MFA (Mükerrer Çok Faktörlü Doğrulama):** Kullanıcının kurumsal ağa veya uygulamaya girerken hem Microsoft Entra ID üzerinde hem de SAP IAS üzerinde art arda iki kez telefona onay düşmesi durumu. Kullanıcı deneyimini bozan bu döngü, IAS üzerindeki Risk-Based kurallar kapatılarak (Pass-Through) engellenir.
* **Ephemeral Certificates (Kısa Ömürlü Sertifikalar):** BTP üzerindeki bir uygulamanın şirket içi S/4HANA sistemine güvenli erişimi sırasında, SAP Cloud Connector tarafından statik şifrelerin yerine dinamik olarak üretilen ve sadece tek bir oturum/istek için (maksimum 2 dakika) geçerli olan X.509 istemci sertifikaları.
* **Grace Period (Geçiş Tolerans Süresi):** Görevi veya departmanı değişen (Mover) bir çalışanın, eski işlerini devredebilmesi ve ay sonu kapanışlarını tamamlayabilmesi amacıyla AD üzerinde her iki departman grubunda birden tutulduğu kontrollü çakışma süresi (örn. 10-15 gün).
* **Identity Federation (Kimlik Federasyonu):** Kullanıcının kimliğinin ve oturumunun, şifre paylaşımı yapılmaksızın güven ilişkisi kurulmuş sistemler arasında kriptografik olarak imzalanmış token'lar (SAML Assertion veya OIDC ID Token) aracılığıyla tanınması süreci.
* **Identity Provisioning (Kimlik ve Yetki Provizyonu):** Kullanıcı hesaplarının, özniteliklerinin ve güvenlik gruplarının kaynak sistemlerden okunarak hedef sistemlerin veritabanına SCIM REST API çağrılarıyla fiziksel olarak yazılması, güncellenmesi veya kilitlenmesi işlemi.
* **Orphan Account (Yetim Hesap):** Şirketten ayrılmış (Leaver) veya pozisyonu değişmiş çalışanların hedef sistemlerde (S/4HANA, BTP) unutulmuş, kilitlenmemiş ve sahipsiz kalmış hesapları. Siber saldırganların en sık istismar ettiği güvenlik zafiyetidir.
* **Principal Propagation (Kimlik İletimi):** Kullanıcının kimliğinin, istemci tarayıcısından buluttaki BTP uygulamasına ve oradan güvenli tünel üzerinden arka uçtaki S/4HANA sistemine kadar hiçbir statik servis hesabı şifresine bulaşmadan, kullanıcının kendi adına X.509 sertifikasıyla taşınması mekanizması.
* **Request-Based Access (Talep Tabanlı Yetki):** Pozisyonla birlikte otomatik gelmeyen; belirli bir proje, istisnai finansal limit artırımı veya acil müdahale (Firefighter) için yönetici onayı ve SoD kontrolü gerektiren geçici yetkiler.
* **Shadow User (Gölge Kullanıcı):** Kullanıcıların BTP alt hesaplarına veya hedef sistem veritabanlarına gereksiz yere ön-provizyonla kopyalanması sonucu oluşan, zamanla güncelliğini yitiren gereksiz kullanıcı kayıtları. JIT Federation modeli gölge kullanıcı oluşumunu engeller.
* **Simulation Mode (Simülasyon Modu):** SAP IPS üzerinde yapılan kural ve mapping değişikliklerinin canlı sistemlere yazmadan önce çalıştırılarak; eklenecek, kilitlenecek ve değiştirilecek kullanıcı sayılarını CSV raporuyla denetleme imkanı veren emniyet modu.
* **Single Source of Truth (Tek Güven Kaynağı):** Kurumsal verinin yalnızca tek bir sistemden doğması ve diğer tüm sistemlerin bu kaynağı referans alması ilkesi. Çalışan kimlik yaşam döngüsünde tek güven kaynağı SAP SuccessFactors'tür.
* **Zero-Touch Provisioning (El Değmeden Provizyon):** İşe giriş, terfi veya ayrılış süreçlerinde IT veya Basis ekiplerine hiçbir destek bileti (ticket) açılmadan, İK sistemindeki olayın tetiklemesiyle tüm sistemlerde hesapların saniyeler içinde otomatik açılması ve kapatılması.

---

## 15. ROL İSİMLENDİRME STANDARTLARI, YAŞAM DÖNGÜSÜ VE İLERİ SEVİYE IAM ARAŞTIRMALARI

### 15.1. SAP Üzerinde Standartlaştırılmış Rol Adlandırma Kuralları (PFCG Naming Conventions)
SAP S/4HANA üzerinde teknik rol ismi (`AGR_NAME`) azami **30 karakter** ile sınırlıdır. Bu kısıt nedeniyle kurumsal standart sözdizimi şu biçimde uygulanır:
`Z:<KAPSAM>_<MODÜL>_<TİP>_<İŞLEV/UNVAN>_<ORGANİZASYON>`

* **Kapsam:** `S4` (S/4HANA), `BT` (BTP), `BW` (BW/4HANA).
* **Modül:** `MM`, `FI`, `CO`, `SD`, `PP`, `PM`, `BC` vb.
* **Rol Tipleri:**
  * `S` (Single Role - Tekil Fonksiyonel Rol): İşlem kodları ve yetki nesnelerini barındıran temel yetki birimi.
  * `D` (Derived Role - Türetilmiş Rol): Menü ve yetkileri ana rolden alan, sadece organizasyon seviyeleri (`$BUKRS`, `$WERKS`) farklılaştırılmış rol.
  * `C` (Composite Role - Birleşik Rol): Birden çok tekil/türetilmiş rolü tek isim altında paketleyen rol.
  * `M` (Master / Template Role - Şablon Rol): Yalnızca türetilmiş rollere miras veren, kullanıcıya atanmayan şablon rol.
* **Fiori Rol Standartları:** `Z:BC_<MODÜL>_<İŞLEV>` (İş Kataloğu) ve `Z:BR_<MODÜL>_<UNVAN>` (İş Rolü / Sayfa Düzeni).
* **BTP Rol Koleksiyonu Standardı:** `<HİZMET>_<FONKSİYON>_<SEVİYE>` (Örn: `BTP_SAC_BI_ANALYST`).
* **Entra ID Grup Standardı:** `SG_SAP_<HEDEF>_<MODÜL>_<İŞLEV>_<KISIT>` (Örn: `SG_SAP_S4P_MM_BUYER_1000`).

### 15.2. Rol Gruplarının Varsayılan Atanma Değerleri (Default Values)
* **SU01 / AGR_USERS:** Varsayılan başlangıç tarihi `SY-DATUM`, bitiş tarihi `31.12.9999`'dur. Kullanıcı tipi standartta `A` (Dialog) atanır.
* **Kullanıcı Parametreleri (SU3 / USR05):** `BUK` (Şirket Kodu), `WRK` (Tesis) gibi değerler standart SCIM protokolü ile doğrudan yazılamaz; bu veri kısıtları kullanıcı parametresiyle değil, türetilmiş rollerle (`Derived Roles`) garanti edilir.
* **IPS Default Transformations:** Boş gelen alanlar için `defaultValue` kuralı işletilir (`preferredLanguage: "tr"`, `timezone: "Europe/Istanbul"`).
* **BTP Varsayılan Koleksiyonlar (Default Role Collections):** Kimliği doğrulanan her kullanıcıya gruptan bağımsız verilen temel BTP izinleri (`Subaccount User`, `Launchpad_Viewer`).

### 15.3. Çift Yönlü Aktarım Deneyleri ve Mock Kullanıcı Tasarımı
* **Döngüsel Senkronizasyon (Circular Sync Loop) Koruması:** Entra ID tekil kimlik kaynağı (Source) kalmalı, S/4HANA ise hedef (Target) olmalıdır. Karşılıklı kullanıcı yazma kurgusu döngüsel çakışmaya yol açtığı için geri yönde yalnızca rol envanteri okuma izin verilir.
* **Mock Kullanıcı Profilleri:**
  * `TEST_MOCK_USER_01` (Standart Alım): Day-1 birthright test senaryosu.
  * `TEST_MOCK_USER_02` (Departman Değişimi): Mover eski rolün geri alınıp yeni rolün atanması senaryosu.
  * `TEST_MOCK_USER_03` (Acil Müdahale): `Z_FF_*` filtresinin doğrulanması senaryosu.
* **Doğrulama Metodu:** Quality/Test subaccount'unda IPS `Simulation Mode` (Dry-Run) tetiklenerek hedef sistemde beklenen atamalar CSV üzerinden doğrulanır.

### 15.4. Kayıt Defteri Detayları ve Revizyon Bazlı Takip
* **SAP Veritabanı Değişiklik Belgeleri:**
  * `CDHDR` (Change Document Header): Değişiklik numarası, işlemi yapan servis kimliği, tarih ve saat.
  * `CDPOS` (Change Document Position): `AGR_USERS` tablosundaki `AGR_NAME` alanının eski (`VALUE_OLD`) ve yeni (`VALUE_NEW`) değerleri.
  * `USH02` ve `USH04`: Oturum kilit geçmişi ve yetki profil revizyon kopyaları.
* **SUIM Denetim Raporları:** `RSUSR100N` (Kullanıcı bazlı yetki değişiklikleri) ve `RSUSR100U` (Rol nesne değişiklikleri).
* **Noktasal Zamanlı Denetim (Point-in-Time Audit):** Belirli bir tarihteki yetkilerin doğrulanması için SF pozisyon geçmişi, Entra ID grup logları ve `AGR_USERS` tarih aralığı korele edilir.

### 15.5. Rol Grubu veya Unvanın Pasifleştirilmesi ve Emekli Edilmesi (Deprecation)
* **Pasifleştirme (Geçici):** Yeni personele atamanın durdurulması, mevcut yetkilerin dondurulması.
* **Emekli Etme (Kalıcı):** Rolün aşamalı tasfiyesi ve sistemden silinmesi.
* **4 Aşamalı Emeklilik Protokolü:**
  1. *İşaretleme:* Rol tanımına `[DEPRECATED]` etiketi eklenir ve yeni atamalara kilitlenir.
  2. *Geçerlilik Bitişi:* `AGR_USERS` tablosunda aktif kullanıcıların `TO_DAT` değeri bugünün tarihine (`SY-DATUM`) çekilir.
  3. *Dizin Tasfiyesi:* AD grubu `SG_SAP_RETIRED_*` olarak yeniden adlandırılır; IPS kaynak filtresine `not (displayName startswith 'SG_SAP_RETIRED_')` eklenir.
  4. *Karantina ve Silme:* En az 90 gün karantina süresi beklenir; iş aksaması yaşanmazsa rol arşivlenerek sistemden silinir.

---

## 16. KONSOLİDE RESMİ STANDARTLAR, RFC'LER VE KAYNAKÇA TABLOSU

| Standart / Kaynak | Resmi Kod / URL | Açıklama / Kapsam |
|:---|:---|:---|
| **SAP KBA 2791410** | [me.sap.com/notes/2791410](https://me.sap.com/notes/2791410) | SuccessFactors Basic Authentication sonlandırılması ve IAS zorunluluğu (13 Kasım 2026). |
| **SAP Note 3048590** | [me.sap.com/notes/3048590](https://me.sap.com/notes/3048590) | SuccessFactors ile Identity Authentication entegrasyonu rehberi. |
| **SAP Note 2656910** | [me.sap.com/notes/2656910](https://me.sap.com/notes/2656910) | Employee Central: Concurrent Employment ve Global Assignment Guide. |
| **SAP Note 2570961** | [me.sap.com/notes/2570961](https://me.sap.com/notes/2570961) | Customer Vendor Integration (CVI) ve S/4HANA BP-User senkronizasyonu. |
| **SAP Note 2753435** | [me.sap.com/notes/2753435](https://me.sap.com/notes/2753435) | SAP Cloud Identity Services tenant modeli (1 Test + 1 Production). |
| **SAP Identity Provisioning Help** | [help.sap.com/docs/identity-provisioning](https://help.sap.com/docs/identity-provisioning) | IPS Transformations, Properties (`ips.delete.threshold`) ve SCIM API motoru. |
| **SAP Identity Directory API** | [api.sap.com/api/IdentityDirectory](https://api.sap.com/api/IdentityDirectory/overview) | SAP resmi SCIM 2.0 şeması: `urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid`. |
| **SAP BTP Security Guide** | [help.sap.com/docs/btp/security](https://help.sap.com/docs/btp/sap-business-technology-platform/security) | XSUAA Role Collection Mapping, OIDC Trust Configuration, Ephemeral Principal Propagation. |
| **SAP CAP Authorization** | [cap.cloud.sap/docs/guides/security/authorization](https://cap.cloud.sap/docs/guides/security/authorization) | CDS seviyesinde deklaratif ABAC (`@restrict: where companyCode = $user.companyCode`). |
| **SAP Cloud Connector Guide** | [help.sap.com/docs/connectivity](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/principal-propagation) | X.509 Short-Lived Certificate ile On-Premise ABAP Principal Propagation. |
| **SAP ABAP Platform PFCG** | [help.sap.com/docs/ABAP_PLATFORM](https://help.sap.com/docs/ABAP_PLATFORM) | Role Administration - Derived Roles Concept in PFCG. |
| **NIST SP 800-207** | [doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207) | Zero Trust Architecture (PDP/PEP mimarisi, Explicit Verification). |
| **NIST SP 800-63B** | [pages.nist.gov/800-63-3/sp800-63b.html](https://pages.nist.gov/800-63-3/sp800-63b.html) | Digital Identity Guidelines: Authentication and Lifecycle Management. |
| **IETF SCIM 2.0 (RFC 7643 / 7644)** | [datatracker.ietf.org/doc/html/rfc7643](https://datatracker.ietf.org/doc/html/rfc7643) | System for Cross-domain Identity Management: Core Schema & Protocol (Append/PATCH syntax). |
| **Microsoft Entra ID SCIM** | [learn.microsoft.com/entra](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups) | Graph API SCIM entegrasyonu ve Dinamik Kullanıcı Grupları. |
| **ISO/IEC 27001:2022** | Kontrol 8.15 | Information Security Management - Logging, Monitoring and Immutable Audit Trail. |
