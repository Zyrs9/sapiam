---
title: Güvenlik, Denetim ve Break-Glass Protokolü
tags:
  - security
  - audit
  - break-glass
  - compliance
  - gdpr
  - kvkk
  - iso27001
status: approved
version: "3.0"
updated: 2026-10-19
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Güvenlik, Denetim ve Break-Glass Protokolü

Bu belge, SAP IAM otomasyonunda felaket senaryolarına karşı alınacak operasyonel önlemleri, yangın söndürücü (Firefighter) istisna kurallarını, kasa (Break-Glass) acil durum prosedürlerini ve KVKK / GDPR / ISO 27001 uyum mimarisini belgeler.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Stratejik Çerçeve: [[01 - Kurumsal Strateji, Olgunluk ve ROI]]
* Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]

---

## 1. Sistemin Aksamaması İçin Hedef Sistemde Tanımlanması Gereken Koruyucu Parametreler

Canlıya geçiş sürecinde sistemin aksamaması ve beklenmeyen yetki kayıplarının yaşanmaması için hedef sistemlerde önceden tanımlanması faydalı görülen 4 temel koruyucu parametre bulunmaktadır:

### 1. Mevcut Yetkilerin Silinmesini Engelleyen Yapılandırma (`authoritative = false`)
* **Neden Şart?** Canlıya geçiş anında otomasyon doğrudan ezici (authoritative) modda açılırsa; sistem, Active Directory gruplarında tanımlı olmayan ancak çalışanın S/4HANA'da işini yapabilmesi için elle verilmiş tüm geçmiş yetkilerini anında silebilir.
* **Çözüm:** İlk etapta sistem sadece AD'den gelen **yeni yetkileri eklemeli**, içeride elle verilmiş mevcut rollere **dokunmamalıdır**.
* **Teknik Parametre:** `s4hana.authoritative.role.management = false` (Geçiş fazı boyunca).

### 2. Toplu Silinme veya Kilitlenmeleri Durduran Eşik Değeri (`delete.threshold = 10`)
* **Neden Şart?** IT tarafındaki bir script hatasında veya AD'deki bir organizasyonel birim (OU) kazara taşındığında, sistem personeli toplu olarak işten ayrılmış (Leaver) algılayabilir.
* **Çözüm:** Tek bir senkronizasyon döngüsünde kullanıcıların %10'undan fazlasını kilitlemeye veya silmeye kalkan herhangi bir işlemde sistem otomatik olarak kendini durdurmalıdır.
* **Teknik Parametre:** `ips.delete.threshold = 10` [^1].

### 3. Acil Müdahale ve Özel Yetki Filtresi (`Z_FF_*` / `Z_EXC_*`)
* **Neden Şart?** Sistem canlıya geçtiğinde ilk haftalarda acil yetki ihtiyaçları ve kriz anları doğabilir. SAP Basis danışmanlarının açacağı geçici kurtarma yetkilerinin otomasyon tarafından kaldırılmaması gerekir.
* **Çözüm:** Başı `Z_FF_` veya `Z_EXC_` ile başlayan roller otomasyon kapsamının dışında tutulmalıdır.
* **Teknik Parametre:** 
  ```properties
  ips.target.options.role.exclude.filter = (Z_FF_.*|Z_EXC_.*)
  ```
  Bu regex filtresi sayesinde acil durum rolleri IPS tarafından okunmaz, değiştirilmez veya silinmez; yalnızca kurumsal GRC / Firefighter modülü tarafından yönetilir [^2].

### 4. Değişmez Ortak Anahtar Alanı (`employeeId`)
* **Neden Şart?** Eşleştirmeyi e-posta veya kullanıcı adı üzerinden yapmak evlilik, unvan değişimi veya e-posta formatı güncellemelerinde veri eşleşmelerinin bozulmasına yol açabilir.
* **Çözüm:** Sistemler arası kimlik bağının SuccessFactors'teki Personel Numarası (`personIdExternal`) ile AD'deki `employeeID` alanı üzerinden tekil anahtar olarak kurgulanması ve S/4HANA `externalId` alanına yazılması sağlıklı olacaktır.

---

## 2. Destekleyici Güvenlik ve Operasyonel IPS Parametreleri

| Parametre Adı | Önerilen Değer | Amacı ve Kritik Rolü |
|:---|:---|:---|
| **`ips.delete.threshold`** | `10` | **Toplu Silinme ve Kilitlenmeyi Önleme:** Toplu kaza durumunda senkronizasyonu durdurarak hedef sistemi korur. |
| **`s4hana.authoritative.role.management`** | `false` *(Geçiş)*<br>`true` *(Canlı)* | Mevcut manuel rollerin korunması / ezilmesi kontrolü. |
| **`ips.trace.failed.entity.content`** | `true` | Hatalı kayıtların payload'unu loglara yazarak troubleshooting sürecini hızlandırır (KVKK uyumlu maskeleme ile). |
| **`ips.job.bulk.page.size`** | `100` | SCIM REST çağrılarında network yükünü ve rate-limit aşımını engellemek için paket büyüklüğünü sınırlar. |
| **`entra.id.filter.groups`** | `displayName startswith 'SG_SAP_'` | **Graph API Throttling Koruması:** Entra ID üzerindeki binlerce grubun çekilerek Graph API paket sınırlarını aşmasını ve IPS timeout hatalarını engeller. |
| **`SU01 Alias = employeeID`** | Pre-Migration Şartı | **Brownfield Çakışma Önleme:** Mevcut S/4HANA kullanıcılarının `externalId` alanına personel numarası yazılarak SCIM 409 Conflict hataları önlenir. |

---

## 3. Olağanüstü Durum Yönetici Hesapları (Break-Glass Protocol)

* **Senaryo:** Microsoft Entra ID veya SAP Cloud Identity Services genelinde küresel bir bulut kesintisi (Cloud Outage) yaşanması veya OIDC/SAML federasyon sertifikasının süresinin dolması.
* **Müdahale Prosedürü (Break-Glass):**
  1. SAP S/4HANA ve SAP BTP subaccount seviyelerinde federasyondan bağımsız, doğrudan yerel kullanıcı veritabanında tanımlı **acil durum süper yönetici hesapları** (`BREAK_GLASS_ADMIN`) barındırılır [^3].
  2. Bu hesapların şifreleri insan belleğinde tutulmaz; kurumsal ayrıcalıklı erişim kasasında (**CyberArk Enterprise Password Vault** veya **Azure Key Vault HSM**) kilitli tutulur.
  3. Kasa şifresi yalnızca CISO ve IT Direktörü'nün çift onayı (Dual-Control / Four-Eyes Principle) ile açılabilir. Kasadan şifre çekildiği saniye SOC (Security Operations Center) ekibine en yüksek seviye (P1) güvenlik alarmı üretilir.

---

## 4. Simülasyon Modu (Simulation Mode) ve Geri Alma (Rollback) Güvencesi

Yeni bir yetki kuralı veya IPS JSON dönüşümü canlı ortama alınmadan önce kesinlikle doğrudan çalıştırılmaz:

1. **Simulation Job:** IPS üzerinde hedef sistem için **"Run Simulation"** seçeneği tetiklenir [^4].
2. **Etki Analizi Raporu:** Simülasyon sonucunda üretilen CSV dosyası incelenir:
   * Kaç kullanıcı güncellenecek?
   * Kaç kullanıcı kilitlenecek (`active = false`)?
   * Hangi roller eklenecek / silinecek?
3. **Rollback Stratejisi:** Beklenmeyen bir toplu değişiklik tespit edilirse, JSON transformation versiyon geçmişinden (Version History) tek tıkla önceki stabil sürüme dönülür.

---

## 5. Regülasyon ve Veri Gizliliği Uyumu (KVKK / GDPR / ISO 27001)

Sistemler arasında kullanıcı verisi akarken yasal uyumluluğu sağlayan 3 temel güvenlik ilkesi:

### A. Veri Asgarileştirmesi (Data Minimization - GDPR Madde 5(1)(c) / KVKK Madde 4)
SAP IPS, Active Directory'den kullanıcının ev adresini, özel telefonunu, medeni durumunu veya özel notlarını **kesinlikle çekmez** [^5]. Sadece kimlik doğrulama ve yetkilendirme için zorunlu olan 5 temel alan taşınır:
* `userName` (Kullanıcı Adı)
* `email` (İş E-postası)
* `employeeId` (Tekil Sicil / Correlation ID)
* `department` (Organizasyonel Birim)
* `groups` (`SG_SAP_*` Güvenlik Grupları)

### B. Değişmez Denetim İzi (Immutable Audit Trail - ISO/IEC 27001:2022 Kontrol 8.15)
Kimin hangi güvenlik grubuna ne zaman girdiği, hangi SAP rolünü ne zaman aldığı ve yetkinin ne zaman geri alındığı bilgisi **SAP Cloud Identity Services Audit Log** servisinde saklanır [^6].
* Loglar kriptografik olarak imzalanır; sistem yöneticileri dahil hiç kimse tarafından değiştirilemez veya silinemez.
* SOX, BDDK ve Bağımsız Denetim süreçlerinde doğrudan yasal delil olarak sunulur.

### C. MFA Çakışma Önleme Kuralı (Single-IdP MFA Policy)
Kullanıcı deneyimini bozan ve destek çağrılarını artıran çift faktör döngüsünü (Double-MFA) engellemek amacıyla:
* **Karar:** Çok Faktörlü Doğrulama (MFA) yalnızca kurumsal IdP olan **Microsoft Entra ID Conditional Access** üzerinde zorunlu kılınır [^7].
* **Uygulama:** SAP IAS üzerindeki Risk-Based MFA devre dışı bırakılır; IAS yalnızca yetki iddialarını taşıyan bir OIDC Proxy olarak çalışır.

---

## Resmi Referanslar ve Kaynakça

[^1]: **SAP Help Portal - Identity Provisioning:** *List of Properties (`ips.delete.threshold`)*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties](https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties)
[^2]: **SAP Help Portal - GRC:** *Emergency Access Management (Superuser Privilege Management / Firefighter)*. [https://help.sap.com/docs/SAP_ACCESS_CONTROL](https://help.sap.com/docs/SAP_ACCESS_CONTROL)
[^3]: **Microsoft Learn - Entra Security:** *Manage emergency access accounts in Microsoft Entra ID*. [https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access)
[^4]: **SAP Help Portal - Identity Provisioning:** *Simulate Provisioning Jobs*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/simulate-provisioning-jobs](https://help.sap.com/docs/identity-provisioning/identity-provisioning/simulate-provisioning-jobs)
[^5]: **European Parliament and Council:** *Regulation (EU) 2016/679 (GDPR), Article 5(1)(c) - Data Minimisation*. [https://gdpr-info.eu/art-5-gdpr/](https://gdpr-info.eu/art-5-gdpr/)
[^6]: **International Organization for Standardization:** *ISO/IEC 27001:2022 Information security, cybersecurity and privacy protection - Control 8.15 (Logging)*. [https://www.iso.org/standard/27001](https://www.iso.org/standard/27001)
[^7]: **NIST SP 800-63B:** *Digital Identity Guidelines: Authentication and Lifecycle Management*. [https://doi.org/10.6028/NIST.SP.800-63b](https://doi.org/10.6028/NIST.SP.800-63b)
