---
title: Çekirdek Mimari ve JML Yaşam Döngüsü
tags:
  - architecture
  - btp
  - s4hana
  - jml
  - jit-provisioning
  - abac
status: approved
version: "3.0"
updated: 2026-10-19
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Çekirdek Mimari ve JML Yaşam Döngüsü

Bu belge, SAP ekosistemindeki kimlik doğrulama, runtime yetkilendirme modellerini, Joiner-Mover-Leaver (JML) uç durumlarını (edge cases) ve veri kısıtlama (ABAC vs. Derived Roles) mimarisini teknik derinliğiyle belgeler.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Stratejik Çerçeve: [[01 - Kurumsal Strateji, Olgunluk ve ROI]]
* Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Taktiksel Saha Rehberi: [[07 - Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi]]

---

## 1. BTP Yetkilendirme Stratejisi: JIT Token Federation vs. IPS Direct Provisioning

SAP BTP üzerinde yetkilendirmenin nasıl işletileceği kurumsal mimarinin en kritik yol ayrımıdır. Sistem kirliliğini (Shadow Users) engellemek ve canlı güncelleme sağlamak için **Hibrit Model** uygulanır [^1]:

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

### Yol A (Önerilen - Uygulama Son Kullanıcıları): Just-In-Time (JIT) OIDC Federation
* **Çalışma Prensibi:** BTP üzerindeki SAP Fiori Launchpad, SAP Build Workzone veya CAP (Cloud Application Programming Model) uygulamalarına erişen son kullanıcılar **IPS ile BTP'ye kopyalanmaz**.
* **Mekanizma:**
  1. Kullanıcı SAP IAS üzerinden oturum açar (Entra ID kurumsal kimliği ile).
  2. IAS, kullanıcının üye olduğu `SG_SAP_*` gruplarını OIDC ID Token / SAML Assertion içine `Groups` claim'i olarak yazar [^2].
  3. BTP Subaccount seviyesindeki **Role Collection Mapping** motoru (XSUAA), gelen token'daki grupları eşleştirir ve kullanıcıya runtime'da anlık olarak rol koleksiyonunu (Role Collection) bağlar.
* **Avantajı:** BTP kullanıcı veritabanında gereksiz kayıt (Shadow User) oluşmaz; AD'de grup değiştiği anda bir sonraki oturumda BTP yetkileri saniyesinde güncellenir.

### Yol B (İstisnai - Yönetici ve Servis Hesapları): IPS Direct Provisioning
* **Kullanım Alanı:** Kullanıcının henüz sisteme hiç oturum açmadan (Day-0) BTP Subaccount Administrator, Cloud Connector Administrator olması gerektiğinde veya arka planda çalışacak Batch Job kimlikleri için IPS'in `SAP BTP Subaccount (Cloud Foundry)` hedef bağlayıcısı üzerinden XSUAA REST API ile ön-provizyon yapılır (Bkz: [[04 - SAP IPS JSON Kod Spesifikasyonu#d-hedef-sistem-sap-btp-subaccount-xsuaa--role-collection-target]]).

### 1.1. SCIM 2.0 Şema ve Dönüşüm Mantığı Denetim Kriterleri

Kurumsal kimlik doğrulama (Authentication) ile arka plan yetkilendirme/provizyon (Authorization & Provisioning) katmanlarının mimari ayrımı ve SCIM 2.0 standart uyumu şu 3 teknik sütunla güvenceye alınır:

1. **Artımlı Senkronizasyon (Delta Sync):** Kaynak ve hedef dönüşümlerde `entityIdExternal`, `groupIdExternal` ve `entityIdTargetSystem` değişkenlerinin tanımlanmış olması, her döngüde tüm dizinin baştan taranmasını engelleyerek sadece değişen kayıtların işlenmesini güvence altına alır.
2. **Resmi SAP Kimlik Şeması:** IAS hedef sisteminde küresel kullanıcı kimliği için resmi `urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid` şemasının kullanılması, sistemler arası tekil anahtar bütünlüğünü sağlar.
3. **Dizi Ekleme (Array Append) Standardı:** S/4HANA ve BTP hedef sistemlerinde çoklu rol atamaları için `$.roles[+].value` ve `$.roleCollections[+].value` sözdiziminin `preserveArrayWithSingleElement: true` parametresiyle birlikte kullanılması RFC 7643/7644 SCIM 2.0 standardına tam uyumludur.

---

## 2. İK Yaşam Döngüsündeki Uç Senaryolar (JML Edge Cases)

Kurumsal operasyonlarda kimlik otomasyonunu en sık sekteye uğratan 4 karmaşık senaryo ve çözüm tasarımı:

### A. Çift Şapkalılık / Eşzamanlı İstihdam (Concurrent Employment)
* **Problem:** Bir çalışan hem Türkiye şirketinde Finans Direktörü, hem de Hollanda iştirakinde Yönetim Kurulu Üyesidir. SuccessFactors Employee Central üzerinde tek bir `personIdExternal` altında iki farklı `userId` ve iki farklı istihdam sözleşmesi (employment contract) bulunur [^3]. Ancak çalışanın Active Directory'de yalnızca **tek bir hesabı** (`sAMAccountName`) vardır.
* **Mimari Çözüm:**
  * Entra ID üzerinde kullanıcıya her iki sözleşmesini temsil eden güvenlik grupları atanır (`SG_SAP_S4H_TR_FIN_DIR` ve `SG_SAP_S4H_NL_BOARD`).
  * SAP BTP ve S/4HANA tarafında yetkiler **kümülatif (birleştirilmiş)** olarak yüklenir.
  * Eğer S/4HANA tarafında mevzuat/denetim gereği iki görevin kesinlikle ayrı kullanıcı kimlikleriyle (`AHMET_TR` ve `AHMET_NL`) yapılması yasal zorunluluksa; IT birimi AD tarafında ikinci bir teknik hesap açmalı ve bu hesap SF'teki ilgili `userId` ile ilişkilendirilmelidir.

### B. Tekrar İşe Alım (Rehire)
* **Problem:** Şirketten 2 yıl önce ayrılmış bir personel tekrar işe başladığında sistem kayıtlarının çakışması veya eski yetkilerin canlanması riski.
* **Mimari Çözüm:**
  * SuccessFactors çalışanın eski `personIdExternal` anahtarını korur.
  * AD tarafında eski hesap yeniden aktive edilir (`accountEnabled = true`).
  * IPS, S/4HANA'da kilitli bekleyen eski `SU01` hesabını `externalId` eşleşmesi sayesinde bulur. Yeni bir kullanıcı açmaya çalışıp hata vermez; mevcut hesabı **Un-lock** eder ve eski departmanından kalan tüm bayat rolleri temizleyerek yalnızca yeni AD gruplarını yazar.

### C. Şirket İçi Yatay Geçiş ve Çift Yetki Toleransı (Cross-Company Transfer)
* **Problem:** Çalışan 1 Kasım itibarıyla Satınalma'dan Finans'a geçmektedir; ancak ay sonu kapanışlarını tamamlayabilmesi için 15 Kasım'a kadar her iki şirkette/departmanda da işlem yapabilmesi gerekir.
* **Mimari Çözüm:**
  * Active Directory tarafında 15 günlük **"Grace Period" (Tolerans Süresi)** uygulanır. Çalışan 15 gün boyunca her iki güvenlik grubunda da tutulur.
  * 15. günün sonunda IT otomasyonu eski departman grubunu siler. IPS'in bir sonraki artımlı senkronizasyonunda eski roller S/4HANA'dan otomatik olarak düşer.

### D. İsim / Soyisim Değişikliği (Name/Surname Change)
* **Problem:** Evlilik veya yasal nedenlerle çalışanın soyadı değişir. AD'de `userPrincipalName` ve `mail` güncellenir (`ayse.kaya` $\rightarrow$ `ayse.yilmaz`).
* **Mimari Çözüm:**
  * Uçtan uca sistemler birbirine `mail` veya `userName` ile değil; **değişmez (immutable) `employeeId` / `userUuid`** ile bağlıdır [^4].
  * IPS, S/4HANA'daki kaydı `externalId` ile bulduğu için kullanıcının kullanıcı adı veya e-postası değişse bile yetki ilişkisi kopmaz; profil bilgisi sorunsuz güncellenir.

---

## 3. Rol ve Yetki Mimarisi: Yetki Tasarımı (Modül İçi) vs. Dağıtımı (IPS) ve Veri Kısıtı

### A. Mimari İş Bölümü: Yetki Tasarımı (Modül İçi) vs. Yetki Dağıtımı (BTP / IPS Merkezi)

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

### B. Önceden Planlanması Gereken Asgari Kararlar

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

### C. Rol Eşleştirme ve Veri Kısıtı (ABAC vs. Derived Roles) Mimarisi

Standart SCIM 2.0 protokolü ve SAP IPS motoru, S/4HANA içerisindeki bir PFCG rolünün organizasyonel seviyelerini (Şirket Kodu: 1000, Satın Alma Örgütü: 1010 vb.) satır satır enjekte edemez [^5]. Bu kısıt aşağıdaki 2 yöntemle mimari güvenceye alınır:

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

#### 1. S/4HANA için Türetilmiş Roller (Derived Roles) Modeli
S/4HANA yetki ekibi organizasyonel alanları boş olan ana roller (Parent Roles) yerine, ilgili veri kısıtları önceden tanımlanmış **Türetilmiş Roller (Derived Roles)** üretir [^6]:
* `SG_SAP_S4H_FI_GL_1000`  $\rightarrow$  `Z_BR_GL_ACCOUNTANT_1000` (Company Code: 1000)
* `SG_SAP_S4H_FI_GL_2000`  $\rightarrow$  `Z_BR_GL_ACCOUNTANT_2000` (Company Code: 2000)
* `SG_SAP_S4H_MM_BUYER_1010` $\rightarrow$ `Z_BR_PURCHASER_1010` (Purchasing Org: 1010)

#### 2. BTP CAP Uygulamaları için Dinamik Nitelik (ABAC) Modeli
BTP üzerinde geliştirilen CAP (Java / Node.js) uygulamalarında roller statiktir (`FinancialAnalyst`), veri kısıtı ise dinamiktir [^7]:
1. Entra ID'de kullanıcının profilindeki `companyCode` veya `costCenter` attribute'u okunur.
2. IAS üzerinden JWT ID Token içine custom claim/attribute olarak basılır.
3. BTP CAP servis katmanında (`service.cds`) deklaratif güvenlik kuralı işletilir:
   ```cds
   annotate CatalogService.Invoices with @(
       restrict: [
           { grant: 'READ', to: 'FinancialAnalyst', where: 'companyCode = $user.companyCode' }
       ]
   );
   ```

---

## 4. Taktiksel Operasyon ve Saha Uygulama Köprüsü

Sistemin teorik vizyonu ile kod seviyesindeki JSON yapılandırması arasında kalan saha operasyonları (Hangi senaryonun % kaç yapılabilir olduğu, İK/IT/Basis ekiplerinden asgari ne isteneceği, CVI/BP ve grup enflasyonu gibi operasyonel tıkanma noktaları ve %80 otomasyon / %20 manuel sınır stratejisi) detaylı olarak **[[07 - Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi]]** dokümanında modellenmiştir.

---

## Resmi Referanslar ve Kaynakça

[^1]: **SAP Help Portal - BTP Security:** *Establish Trust and Federation Between UAA and Identity Authentication*. [https://help.sap.com/docs/btp/sap-business-technology-platform/establish-trust-and-federation-between-uaa-and-identity-authentication](https://help.sap.com/docs/btp/sap-business-technology-platform/establish-trust-and-federation-between-uaa-and-identity-authentication)
[^2]: **SAP Help Portal - Identity Authentication:** *Configure User Group Assertion for Applications*. [https://help.sap.com/docs/identity-authentication/identity-authentication/configure-user-group-assertion](https://help.sap.com/docs/identity-authentication/identity-authentication/configure-user-group-assertion)
[^3]: **SAP Note 2656910:** *Employee Central: Concurrent Employment and Global Assignment Guide*. [https://me.sap.com/notes/2656910](https://me.sap.com/notes/2656910)
[^4]: **SAP Business Accelerator Hub:** *Identity Directory API - User Schema Extensions (`userUuid`)*. [https://api.sap.com/api/IdentityDirectory/overview](https://api.sap.com/api/IdentityDirectory/overview)
[^5]: **SAP Help Portal - Identity Provisioning:** *Integrating SAP S/4HANA On-Premise / Private Cloud*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-s-4hana](https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-s-4hana)
[^6]: **SAP Help Portal - ABAP Platform:** *Role Administration - Derived Roles Concept in PFCG*. [https://help.sap.com/docs/ABAP_PLATFORM/pfcg_derived_roles](https://help.sap.com/docs/ABAP_PLATFORM)
[^7]: **SAP Cloud Application Programming Model (CAP):** *Authorization and Access Control Guide*. [https://cap.cloud.sap/docs/guides/security/authorization](https://cap.cloud.sap/docs/guides/security/authorization)
