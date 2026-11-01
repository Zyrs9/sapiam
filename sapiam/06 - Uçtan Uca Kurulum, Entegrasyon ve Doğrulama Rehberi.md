---
title: Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi
tags:
  - implementation
  - setup
  - scc
  - entra-id
  - ias
  - ips
  - s4hana
  - testing
status: approved
version: "3.1"
updated: 2026-10-22
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi (5 Fazlı Uygulama Kılavuzu)

SAP ekosisteminde kimlik federasyonu (IAS), kullanıcı/rol provizyonu (IPS) ve yetkilendirme (BTP & S/4HANA) mekanizmalarının uçtan uca devreye alınması; ağ tünellerinden sertifika zincirlerine, SCIM dönüşümlerinden çalışma zamanı güven ilişkilerine kadar 5 ana kurulum fazından oluşur.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* JSON Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Güvenlik ve Koruyucu Parametreler: [[05 - Güvenlik, Denetim ve Break-Glass]]

---

## FAZ 1: Altyapı, Ağ Tüneli ve Ön Hazırlık Kurulumu

Kurulumun bu aşamasında sistemlerin birbirine güvenli erişimini sağlayacak ağ katmanı, tüneller ve dizin izinleri yapılandırılır:

```
[On-Premise / PCE Ağ]                 [Güvenli Tünel]                [SAP BTP Bulut Ağı]
S/4HANA (ICF / RFC)  ◄──────►  SAP Cloud Connector (SCC)  ◄──────►  BTP Subaccount / IPS Engine
```

### 1.1. Microsoft Entra ID Uygulama Kaydı (App Registration)

SAP IPS'in Entra ID Graph API üzerinden kullanıcı ve grupları okuyabilmesi için kurumsal dizinde bir servis kimliği açılır:

1. **App Registration:** Entra ID yönetim portalından `SAP-IPS-Inbound-Connector` adıyla yeni bir uygulama kaydedilir.
2. **API İzinleri (Application Permissions):**
   * `User.Read.All` (Kullanıcı profillerini ve `employeeId` değerini okuma).
   * `GroupMember.Read.All` (Yalnızca güvenlik gruplarını ve grup üyelerini okuma).
3. **Kimlik Doğrulama:** Bir **Client Secret** oluşturulur veya kurumsal PKI altyapısından üretilen bir **X.509 Sertifikası** yüklenir (Önerilen: Sertifika tabanlı kimlik doğrulama).
4. **Çıktılar:** `Tenant ID`, `Client ID` ve `Client Secret / Certificate` değerleri IPS yapılandırması için not edilir.

### 1.2. SAP Cloud Connector (SCC) Kurulumu ve Subaccount Bağlantısı

On-Premise veya Private Cloud S/4HANA sistemlerinin buluta açılması için DMZ veya iç ağda kurulu SCC üzerinde tünel tanımlanır:

1. **Subaccount Tanımlama:** SCC konsolundan BTP Subaccount (Region, Subaccount ID, Subaccount User) eklenir.
2. **S/4HANA Sistem Eşlemesi (Cloud To On-Premise):**
   * *Backend Type:* ABAP System.
   * *Protocol:* HTTPS.
   * *Internal Host:* `s4hana.internal.corp:44300` $\rightarrow$ *Virtual Host:* `s4hana-scim.corp.dummy:443`.
3. **Erişilebilir Kaynaklar (Accessible Resources):**
   * URL Path: `/sap/bc/scim` (Path and all sub-paths).
   * RFC/BAPI kullanılacaksa: BAPI fonksiyon modülü prefixleri (`BAPI_USER_*`).

### 1.3. S/4HANA Tarafındaki Servis ve Sertifika Aktivasyonları

1. **ICF Servis Aktivasyonu (`SICF`):** `/default_host/sap/bc/scim` ağacı aktif hale getirilir.
2. **Sertifika ve SSL Yapılandırması (`STRUST`):** SCC'nin sanal anahtar sertifikası S/4HANA `SSL Server Standard` PSE içine güvenilen sertifika (Trust Certificate) olarak eklenir.
3. **Teknik Kullanıcı:** IPS'in SCIM çağrılarını karşılayacak en düşük yetkili bir sistem kullanıcısı (`SVC_IPS_SCIM`) tanımlanır ve `SAP_BC_SCIM_ADMIN` yetki rolü atanır.

> [!NOTE]
> **Operasyonel Not (Lisans / FUE):**  
> S/4HANA tarafında kullanıcı açılırken varsayılan lisans tipinin kontrolsüz atanmasını önlemek adına, `SU01` üzerinde varsayılan sözleşmesel kullanıcı tipi (*Professional*, *Functional* veya *Productivity*) modül yetki ekibiyle önceden mutabık kalınmalıdır.

### 1.4. Mevcut (Brownfield) S/4HANA Kullanıcılarının Ön Eşitlenmesi (Pre-Migration Data Cleanup)

Canlıya geçiş öncesinde SAP Basis ekibi mevcut aktif `SU01` kullanıcılarının `externalId` / `Alias` alanlarını İK personel numarasıyla (`employeeID`) eşitlemelidir (`SU10` veya toplu ABAP güncelleme raporu ile).

* **Neden Zorunlu?** Mevcut kullanıcıların `externalId` alanı boş bırakılırsa, IPS ilk çalıştığında bu kullanıcıları SCIM üzerinden eşleştiremez. Aynı `userName` ile yeni bir kullanıcı yaratmaya çalışır ve sistem `HTTP 409 Conflict` (kullanıcı zaten mevcut) hatası üreterek senkronizasyonu durdurur.
* **Operasyonel Çıktı:** Tüm aktif personelin `SU01 Alias` = `employeeID` eşleşmesi tamamlanmadan Faz 3'teki hedef sistem aktif edilmemelidir.

---

## FAZ 2: Kimlik Federasyonu ve Güven (Trust) Kurulumu

Bu fazda kullanıcıların giriş yaparken şifre girecekleri kurumsal SSO hattı oluşturulur:

```
[Kullanıcı Tarayıcısı] ──► [BTP App / Launchpad] ──► [SAP IAS (Proxy)] ──► [Entra ID (MFA / IdP)]
```

### 2.1. Entra ID $\leftrightarrow$ SAP IAS Güven İlişkisi (Corporate Identity Provider)

1. **IAS Konsoluna Giriş:** `https://<tenant-id>.accounts.ondemand.com/admin/` adresine gidilir.
2. **Corporate IdP Ekleme:** *Identity Providers* $\rightarrow$ *Corporate Identity Providers* $\rightarrow$ *Create* adımı seçilir:
   * Protokol olarak **OpenID Connect (OIDC)** veya **SAML 2.0** seçilir.
   * Entra ID Federasyon Metadata XML'i (veya OIDC Discovery URL'i) yüklenir.
3. **Identity Federation ve Claim Ayarları:**
   * *Unique User Attribute:* `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent` $\rightarrow$ `employeeID`.
   * *Group Claim:* Entra ID'nin gönderdiği `groups` token talebi IAS üzerinde harici grup niteliği olarak karşılanır.
4. **Tek Merkezli MFA Ayarı (Single-IdP MFA):** IAS üzerindeki *Risk-Based Authentication* kuralları kapatılarak tüm koşullu erişim ve MFA sorumluluğu kurumsal Entra ID tarafına devredilir (Pass-Through).

### 2.2. SAP IAS $\leftrightarrow$ BTP Subaccount Güven İlişkisi

1. **BTP Cockpit'e Giriş:** Subaccount $\rightarrow$ *Security* $\rightarrow$ *Trust Configuration* sekmesine gidilir.
2. **Establish Trust:** *Establish Trust* butonuna basılarak bağlı SAP Cloud Identity Services tenant'ı otomatik eşleştirilir (OIDC protokolüyle).
3. **Assertion Attributes Tanımlama:**
   * Attribute `Groups` $\rightarrow$ IAS'tan gelen `groups` claim'ine eşlenir.
   * Attribute `CompanyCode` (ABAC kullanılacaksa) $\rightarrow$ Kullanıcının şirket kodu alanına bağlanır.

---

## FAZ 3: IPS Veri Dağıtım Motorunun Kurulumu ve Hedef Sistemler

Kullanıcıların ve grupların arka planda dağıtılacağı asenkron SCIM boru hattı kurulur:

### 3.1. Kaynak Sistem (Source System) Kurulumu: Entra ID

*IPS Konsolu* $\rightarrow$ *Source Systems* $\rightarrow$ *Create*:
* **System Type:** `Microsoft Entra ID`
* **Properties:**
  * `Type: HTTP`
  * `Authentication: BasicAuthentication / OAuth2ClientCredentials`
  * `entra.id.client.id: <Faz 1'den alınan Client ID>`
  * `entra.id.client.secret: <Faz 1'den alınan Secret>`
  * `entra.id.tenant.id: <Faz 1'den alınan Tenant ID>`
  * `entra.id.filter.groups: displayName startswith 'SG_SAP_'` (Network trafiğini optimize etmek için sunucu taraflı filtre).

### 3.2. Hedef Sistem (Target System 1) Kurulumu: SAP S/4HANA On-Premise / PCE

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

### 3.3. Hedef Sistem (Target System 2) Kurulumu: SAP Cloud Identity Services (Identity Directory)

* **System Type:** `SAP Cloud Identity Services`
* **Properties:**
  * `Type: HTTP`
  * `URL: https://<tenant-id>.accounts.ondemand.com/service/scim`
  * `Authentication: ClientCertificateAuthentication` (IAS üzerinden üretilen teknik API sertifikası).

---

## FAZ 4: Çalışma Zamanı Yetki Eşleme (Role Mapping) Yapılandırması

Modül danışmanları PFCG ve ilgili modül içinde rolleri tasarladıktan sonra bu rollerin kurumsal gruplarla fiziki bağı kurulur:

```
[Active Directory Grubu]                 [Eşleştirme Katmanı]               [Hedef Sistem Rolü]
SG_SAP_S4H_FI_GL_1000     ──► (IPS JSON Mapping)             ──► S/4HANA: Z_BR_GL_ACCOUNTANT_1000
SG_SAP_BTP_DEV_MGR        ──► (BTP Cockpit Role Collection)  ──► BTP: Procurement_Manager_Collection
```

### 4.1. BTP Subaccount Seviyesinde Role Collection Eşleştirmesi

BTP uygulamaları için kullanıcı kopyalaması yapılmaz; anlık eşleme kuralı girilir:

1. **BTP Cockpit** $\rightarrow$ **Security** $\rightarrow$ **Role Collections**.
2. Hedef rol koleksiyonuna tıklanır (Örn: `Procurement_Manager_Collection`).
3. **User Groups** tabında **Add** seçeneğiyle:
   * *Identity Provider:* `SAP Cloud Identity Services`
   * *Name:* `SG_SAP_BTP_PROCUREMENT_MGR` (AD grubuyla harfi harfine aynı).

### 4.2. S/4HANA Tarafında Principal Propagation (X.509 Kural Tanımlama)

BTP uygulamasından S/4HANA'ya gelen çağrılarda kullanıcının kimliğinin korunması için:

1. **İşlem Kodu `CERTRULE`:** S/4HANA üzerinde sertifika eşleme kuralı açılır:
   * *Issuer DN:* SCC kurumsal CA sertifikasının DN bilgisi.
   * *Subject Component:* `CN` (Common Name) $\rightarrow$ `SU01 Alias` veya `User ID` eşleşmesi kuralı tanımlanır.
2. Bu sayede SCC'den gelen dinamik X.509 sertifikası doğrudan doğruya S/4HANA'daki yerel kullanıcı hesabına bağlanır.

---

## FAZ 5: Sistemin Oturması, Test ve Doğrulama Adımları

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
