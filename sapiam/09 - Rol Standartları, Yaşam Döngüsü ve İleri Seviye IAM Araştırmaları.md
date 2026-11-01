---
title: Rol İsimlendirme Standartları, Yaşam Döngüsü ve İleri Seviye IAM Araştırmaları
tags:
  - sap/iam
  - sap/s4hana
  - sap/btp
  - pfcg
  - naming-conventions
  - role-lifecycle
  - audit
  - mock-testing
  - references
  - glossary
status: approved
version: "2.0"
updated: 2026-10-27
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Rol İsimlendirme Standartları, Yaşam Döngüsü ve İleri Seviye IAM Araştırmaları

Bu doküman, SAP S/4HANA, SAP BTP ve kurumsal dizin (Microsoft Entra ID) entegrasyonunda yetki yaşam döngüsünün derinlemesine teknik detaylarını, rol isimlendirme standartlarını, çift yönlü senkronizasyon deneylerini, denetim kayıt defteri mekanizmalarını ve rol/unvan emeklilik prosedürlerini incelemek amacıyla hazırlanmıştır.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* JSON Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Güvenlik & Kasa: [[05 - Güvenlik, Denetim ve Break-Glass]]
* Terimler ve Kısaltmalar: [[08 - Terimler ve Kısaltmalar Sözlüğü (Glossary)]]

Dokümandaki teknik desenler, **SAP Güvenlik Kılavuzları (SAP Security Guidelines)** [^1], **SAP S/4HANA Yetkilendirme Konsepti** [^2], **IETF SCIM 2.0 (RFC 7643 / 7644)** [^3] ve kurumsal denetim standartlarına (SOX 404, ISO/IEC 27001:2022, NIST SP 800-207) [^4] dayandırılmıştır.

---

## 1. SAP Üzerinde Standartlaştırılmış Rol Adlandırma Kuralları (Naming Conventions)

Yetki yönetiminde sürdürülebilirlik, rollerin teknik isminden rolün türünün, etki alanının, modülünün ve veri kısıtının anlaşılabilmesine bağlıdır. Standart bir isimlendirme olmadan merkezi IAM veya IPS otomasyonu yürütülemez.

### 1.1. PFCG Teknik Rol İsimlendirme Sözdizimi (Syntax ve Kısıtlar)

SAP ABAP ve S/4HANA veritabanında teknik rol ismi (`AGR_NAME`) azami **30 karakter** ile sınırlandırılmıştır [^5]. Bu kısıt, isimlendirme kurallarının modüler, standart ve sıkıştırılmış kodlarla tasarlanmasını zorunlu kılar.

```
+-------------------------------------------------------------------------+
|                  SAP PFCG ROL İSİMLENDİRME STANDARDI                     |
|                                                                         |
|   Z : <KAPSAM> _ <MODÜL> _ <TİP> _ <İŞLEV / UNVAN> _ <ORGANİZASYON>     |
|   ──   ───────   ───────   ─────   ───────────────   ──────────────     |
|  Önek   Sistem    SAP       Rol     Fonksiyonel      Veri Kısıtı        |
|  Alanı  Kapsamı   Modülü    Tipi    Görev Tanımı     (Şirket/Tesis)     |
+-------------------------------------------------------------------------+
```

#### Alan Kodlamaları ve Karakter Dağılımı

1. **Önek Alanı (Namespace - 2 Karakter):**
   - `Z:` : Şirket içi geliştirilen ve özelleştirilen müşteri yetki alanı (Önerilen standart) [^6].
   - `Y:` : Alternatif müşteri ad alanı (Çoklu şablon veya şirket birleşmelerinde ayrım için) [^6].
   - `SAP_` : SAP standart fabrika teslimi roller. Bu roller doğrudan kullanıcılara atanmaz; kopyalanarak `Z:` alanına taşınır [^1].

2. **Sistem Kapsamı (2 Karakter):**
   - `S4` : SAP S/4HANA On-Premise / Private Cloud Edition çekirdek sistemi.
   - `BT` : SAP Business Technology Platform (BTP) entegrasyon rolleri.
   - `BW` : SAP BW/4HANA raporlama yetkileri.

3. **Modül Kodu (2 Karakter):**
   - `MM` : Malzeme Yönetimi ve Satınalma.
   - `FI` : Finansal Muhasebe.
   - `CO` : Maliyet Muhasebesi ve Kontrol.
   - `SD` : Satış ve Dağıtım.
   - `PP` : Üretim Planlama.
   - `PM` : Bakım ve Onarım.
   - `HR` : İnsan Kaynakları (HCM / SuccessFactors entegrasyonu).
   - `BC` : Basis ve Sistem Yönetimi.

4. **Rol Tipi Kodlaması (1 Karakter):**
   - `S` (Single Role - Tekil Fonksiyonel Rol): İşlem kodları (T-Code), yetki nesneleri ve alan değerlerini barındıran temel yetki birimi. Menüsü ve yetki veritabanı doğrudan PFCG'de oluşturulur.
   - `D` (Derived Role - Türetilmiş Rol): Menü ve işlem kodlarını ana (Master) rolden miras alan, sadece organizasyon seviyesi değerleri (`$BUKRS`, `$WERKS`, `$EKORG`, `$VKORG`) özelleştirilmiş rol [^7].
   - `C` (Composite Role - Birleşik Rol): Menü ve yetki nesnesi içermeyen, kullanıcıya tek seferde birden çok tekil/türetilmiş rol bağlayan paket rol [^2].
   - `M` (Master / Template Role - Şablon Rol): Kullanıcıya doğrudan atanmayan, sadece türetilmiş rollere miras veren şablon rol [^7].

5. **Fonksiyonel İşlev / Unvan (Azami 10-12 Karakter):**
   - İlgili personelin operasyonel işlevini belirten standart kısaltma (Örn: `BUYER_OP`, `GL_ACCOUNT`, `WH_CLERK`, `APPROVER`).

6. **Organizasyonel Veri Kısıtı (Azami 4-6 Karakter):**
   - Şirket kodu (`BUKRS`), tesis (`WERKS`) veya organizasyon birimi kodu (Örn: `1000`, `2000`, `ALL`) [^7].

#### Standartlaştırılmış PFCG Rol İsimlendirme Örnekleri

| Teknik Rol İsmi | Rol Türü | Modül | Açıklama |
|:---|:---|:---|:---|
| `Z:S4_MM_M_BUYER_TMPL` | Master (Şablon) | MM | Satınalma Uzmanı ana şablon rolü (Kullanıcıya atanmaz). |
| `Z:S4_MM_D_BUYER_1000` | Derived (Türetilmiş) | MM | 1000 Şirket Kodu için türetilmiş Satınalma Uzmanı rolü. |
| `Z:S4_MM_D_BUYER_2000` | Derived (Türetilmiş) | MM | 2000 Şirket Kodu için türetilmiş Satınalma Uzmanı rolü. |
| `Z:S4_FI_S_AP_CLERK` | Single (Tekil) | FI | Satıcı fatura kayıt personeli temel işlem rolü. |
| `Z:S4_MM_C_SR_BUYER_1000` | Composite (Birleşik) | MM | Kıdemli Satınalmacı için tekil ve türetilmiş rollerin toplu paketi. |
| `Z:S4_BC_S_BASIS_ADMIN` | Single (Tekil) | Basis | Sistem yöneticisi teknik destek rolü. |

---

### 1.2. Fiori Rolleri İsimlendirme Standardı (Spaces, Pages ve Catalogs)

SAP Fiori arayüzünde yetkilendirme iki katmandan oluşur: **İş Kataloğu (Business Catalog)** yetki nesnelerini açarken, **İş Grubu / Alanı (Spaces & Pages)** görsel yerleşimi belirler [^8].

- **Fiori İş Kataloğu Rolü (Business Catalog):**
  - Standart Format: `Z:BC_<MODÜL>_<İŞLEV>`
  - Örnek: `Z:BC_MM_PURCHASING` (Satınalma uygulamalarının uygulama ve OData servis kataloglarını içerir).
- **Fiori Sayfa ve Alan Rolü (Spaces & Pages / Business Role):**
  - Standart Format: `Z:BR_<MODÜL>_<UNVAN>`
  - Örnek: `Z:BR_MM_BUYER` (Satınalma uzmanının Fiori Launchpad üzerinde göreceği alan ve sayfa düzenini içerir).

---

### 1.3. SAP BTP Rol Koleksiyonu (Role Collection) Standartları

SAP BTP Subaccount seviyesinde roller tek tek değil, `Role Collection` adı verilen koleksiyonlar altında gruplanarak kullanıcılara atanır [^9].

- **Standart Format:** `<UYGULAMA/SERVİS>_<MODÜL/FONKSİYON>_<SEVİYE>`
- **Örnekler:**
  - `BTP_SAC_BI_ANALYST` : SAP Analytics Cloud analist seviyesi rol koleksiyonu.
  - `BTP_BUILD_PROCESS_APPROVER` : SAP Build Process Automation onaycı koleksiyonu.
  - `BTP_CAP_FIN_VIEWER` : BTP üzerinde koşan Cloud Application Programming (CAP) uygulaması finans izleme rolü.
  - `BTP_ADMIN_SUBACCOUNT` : Subaccount yönetimsel teknik destek koleksiyonu.

---

### 1.4. Microsoft Entra ID / Active Directory Grup İsimlendirme Standardı

Merkezi dizin üzerinden SAP rollerini tetiklemek için kullanılan güvenlik grupları doğrudan hedef sistem ve rol ile eşleşmelidir [^10].

- **Standart Format:** `SG_SAP_<HEDEF_SİSTEM>_<MODÜL>_<İŞLEV>_<KISIT>`
- **Örnekler:**
  - `SG_SAP_S4P_MM_BUYER_1000` $\rightarrow$ S/4HANA Production, MM Modülü, Şirket Kodu 1000 Satınalma Rolü.
  - `SG_SAP_S4P_FI_AP_CLERK` $\rightarrow$ S/4HANA Production, FI Borç Muhasebesi Rolü.
  - `SG_SAP_BTP_SAC_ANALYST` $\rightarrow$ SAP BTP, Analytics Cloud Analist Rol Koleksiyonu.
  - `SG_SAP_ALL_EMPLOYEES` $\rightarrow$ Tüm çalışanlar için geçerli temel ESS (Employee Self-Service) rol grubu.

---

## 2. Rol Gruplarının Varsayılan Atanma Değerleri (Default Assignment Values)

Kullanıcıya veya gruba sistemde rol atanırken devreye giren varsayılan parametreler, tarih sınırları ve nitelik kuralları sistem tutarlılığını sağlar.

### 2.1. SAP S/4HANA SU01 ve AGR_USERS Varsayılan Parametreleri

Kullanıcıya bir PFCG rolü bağlandığında arka planda `AGR_USERS` tablosuna kayıt atılır [^11]. Standart atanma değerleri şunlardır:

```
+-------------------------------------------------------------------------+
|                  AGR_USERS TABLOSU VARSAYILAN DEĞERLERİ                 |
|                                                                         |
|  UNAME (Kullanıcı Adı)   : AHMET.YILMAZ                                 |
|  AGR_NAME (Teknik Rol)   : Z:S4_MM_D_BUYER_1000                         |
|  FROM_DAT (Başlangıç)    : SY-DATUM (Örn: 2026-10-27 - Atama Günü)      |
|  TO_DAT (Bitiş Tarihi)   : 31.12.9999 (Sonsuz Geçerlilik)               |
|  EXCLUDE (Hariç Tutma)   : ' ' (Aktif Kullanım)                         |
|  ORG_FLAG (Org Bayrağı)  : ' ' (Doğrudan Atama)                         |
+-------------------------------------------------------------------------+
```

1. **Geçerlilik Başlangıç ve Bitiş Tarihleri (`FROM_DAT` / `TO_DAT`):**
   - Varsayılan başlangıç tarihi günün tarihidir (`SY-DATUM`).
   - Varsayılan bitiş tarihi `31.12.9999` olarak set edilir [^11].
   - SCIM protokolü üzerinden atamalarda bu değerler JSON yükündeki `validFrom` ve `validTo` niteliklerinden beslenir; boş bırakılırsa hedef sistem varsayılanları devreye girer [^3].

2. **Kullanıcı Tipi (`USTYP` - User Type):**
   - Standart işe girişlerde kullanıcı tipi varsayılan olarak `A` (Dialog) atanır [^12].
   - BTP ve teknik entegrasyonlar için açılan hesaplar `B` (System User) veya `S` (Service User) olarak yapılandırılmalıdır [^12].

3. **Kullanıcı Parametreleri (SU3 / `USR05` Parametreleri):**
   - Kullanıcının işlem kodlarında varsayılan olarak göreceği organizasyon parametreleri:
     - `BUK` : Varsayılan Şirket Kodu (Örn: `1000`).
     - `WRK` : Varsayılan Tesis / Üretim Yeri (Örn: `1010`).
     - `EKO` : Varsayılan Satınalma Organizasyonu (Örn: `1000`).
     - `VKO` : Varsayılan Satış Organizasyonu (Örn: `1000`).
   - **Teknik Kısıt:** Standart SCIM 2.0 çekirdek şeması SU3 parametrelerini doğrudan desteklemez [^13]. Bu nedenle veri kısıtları kullanıcı parametresiyle değil, doğrudan türetilmiş rol (Derived Role) yetki nesnesi üzerinden garanti altına alınır [^7].

---

### 2.2. SAP IPS Varsayılan Nitelik Atama Kuralları (Transformations `defaultValue`)

IPS kaynak sistemden (Entra ID / SuccessFactors) veri çekerken bazı alanların boş gelmesi durumunda sistemin hata vermesini önlemek için `defaultValue` mekanizmasını kullanır [^14].

```json
{
  "sourcePath": "$.department",
  "targetAttribute": "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User:department",
  "defaultValue": "GENEL_MERKEZ",
  "optional": true
},
{
  "sourcePath": "$.preferredLanguage",
  "targetAttribute": "preferredLanguage",
  "defaultValue": "tr",
  "optional": true
},
{
  "sourcePath": "$.timezone",
  "targetAttribute": "timezone",
  "defaultValue": "Europe/Istanbul",
  "optional": true
}
```

- **Koşullu Varsayılan Değer Ataması:**
  Eğer kullanıcının şirket kodu boş gelirse, provizyon işleminin durmaması için geçici bir varsayılan atanabilir veya provizyon `condition` filtresi ile güvenli biçimde beklemeye alınabilir [^14].

---

### 2.3. SAP BTP Varsayılan Rol Koleksiyonu Ataması (Default Role Collections)

SAP BTP Subaccount üzerinde OIDC / SAML 2.0 Trust yapılandırmasında **Default Role Collections** özelliği bulunur [^15]:

- Kullanıcı kurumsal IdP (Entra ID) üzerinden başarıyla doğrulandığı anda, AD gruplarında herhangi bir özel SAP rolü tanımlı olmasa dahi bu varsayılan koleksiyonları otomatik olarak alır.
- **Önerilen Varsayılan BTP Koleksiyonları:**
  - `Subaccount User` : Temel platform oturum açma yetkisi.
  - `Launchpad_Viewer` : Fiori Cloud Portal üzerinde temel ana sayfayı görüntüleme yetkisi.

---

## 3. Modül Üzerinden BTP Üzerine Çift Yönlü Rol Aktarımı İçin Mock Kullanıcı ve Denemeler

Uçtan uca mimaride çift yönlü (Bidirectional) yetki aktarımı, sistemlerin senkron kalmasını sağlarken döngüsel çakışma (Circular Sync Loop) riskini de beraberinde getirir.

```
                    İLERİ YÖN (FORWARD PROVISIONING)
  Entra ID ────────────────────► SAP IPS ────────────────────► S/4HANA & BTP
  (Kullanıcı & Grup)            (SCIM Dönüşüm Motoru)         (PFCG Rolleri &
                                                               Rol Koleksiyonları)

                    GERİ YÖN (REVERSE PROVISIONING)
  S/4HANA & BTP ───────────────► SAP IPS ────────────────────► SAP IAS / Entra ID
  (Rol Envanteri &              (SCIM Okuma & Eşleme)          (Identity Directory /
   Atama Durumu)                                               Gölge Kullanıcılar)
```

### 3.1. Çift Yönlü Senkronizasyon Riskleri ve Mimari Sınırlar

1. **Döngüsel Çakışma (Circular Sync Loop) Riski:**
   - Eğer Sistem A, Sistem B'ye kullanıcı ve rol yazarken; Sistem B de Sistem A'daki aynı kayıtları güncellemeye çalışırsa sonsuz bir senkronizasyon döngüsü oluşur [^16].
   - **Çözüm İlkesi (Single Source of Truth):**
     - Kullanıcı kimlik verisinin ve grup üyeliğinin tek kaynağı **Entra ID / SuccessFactors** olmalıdır.
     - S/4HANA ve BTP sistemleri kullanıcı nitelikleri için yalnızca **Hedef (Target)** olarak konumlandırılmalıdır.
     - Geri yönde aktarım yapılacaksa bu yalnızca **Rol Envanteri Okuma (Role Inventory Discovery)** ile sınırlandırılmalı; kullanıcı yetki atamaları geri yönde ezilmemelidir.

2. **Yetki Sahipliği (`authoritative = false`) Zorunluluğu:**
   - Geri yönde veya çift yönlü senkronizasyonda `authoritative = false` ayarlanmadığı takdirde, S/4HANA üzerinde Basis tarafından elle verilmiş acil durum veya geçici yetkiler silinir [^17].

---

### 3.2. Mock Kullanıcı ve Test Senaryoları Tasarımı

Entegrasyon testlerinde canlı veriyi riske atmamak için 3 farklı mock kullanıcı profili oluşturulmalıdır:

| Mock Kullanıcı | Senaryo Tipi | Test Edilecek Nitelikler | Beklenen Sistem Davranışı |
|:---|:---|:---|:---|
| `TEST_MOCK_USER_01` | Standart Giriş (Day-1 Birthright) | `SG_SAP_S4P_MM_BUYER_1000`<br>`SG_SAP_ALL_EMPLOYEES` | S/4HANA'da hesap açılır; `Z:S4_MM_D_BUYER_1000` ve `Z:S4_BASIS_S_EMPLOYEE_ESS` rolleri bağlanır. |
| `TEST_MOCK_USER_02` | Görev Değişimi (Mover Testi) | MM grubundan çıkarılıp `SG_SAP_S4P_FI_AP_CLERK` grubuna eklenir. | MM rolü geri alınır; FI rolü eklenir. Mevcut SU01 temel parametreleri korunur. |
| `TEST_MOCK_USER_03` | Acil Durum / Filtre Testi | Kullanıcıya S/4HANA'da doğrudan `Z_FF_BASIS` rolü atanır; AD grubunda bu rol yoktur. | IPS çalıştığında `Z_FF_*` filtresi sayesinde acil durum rolü silinmez, korunur [^18]. |

---

### 3.3. SCIM 2.0 Mock JSON Yükleri (Payloads)

SCIM 2.0 standartlarına (RFC 7643 / RFC 7644) ve SAP Identity Directory şemasına uygun mock test yükleri [^3][^19]:

#### 1. Mock Kullanıcı Oluşturma İsteği (POST `/Users`)

```json
{
  "schemas": [
    "urn:ietf:params:scim:schemas:core:2.0:User",
    "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User",
    "urn:ietf:params:scim:schemas:extension:sap:2.0:User"
  ],
  "userName": "TEST_MOCK_USER_01",
  "externalId": "MOCK_PERNR_90001",
  "name": {
    "givenName": "Deneme",
    "familyName": "Kullanici"
  },
  "emails": [
    {
      "value": "deneme.kullanici@sirket.com",
      "type": "work",
      "primary": true
    }
  ],
  "active": true,
  "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
    "employeeNumber": "90001",
    "costCenter": "CC_1000",
    "department": "SATINALMA"
  },
  "urn:ietf:params:scim:schemas:extension:sap:2.0:User": {
    "userUuid": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
    "validFrom": "2026-10-27T00:00:00Z",
    "validTo": "9999-12-31T23:59:59Z"
  }
}
```

#### 2. Mock Grup ve Rol Atama İsteği (PATCH `/Groups`)

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    {
      "op": "add",
      "path": "members",
      "value": [
        {
          "value": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
          "display": "TEST_MOCK_USER_01"
        }
      ]
    }
  ]
}
```

---

### 3.4. Test Metodolojisi ve Simülasyon Doğrulaması

1. **İzole Test Ortamı:** Mock denemeleri doğrudan Canlı (Production) üzerinde değil; Quality (QA/Test) subaccount ve S/4HANA Test istemcisinde (Client) yürütülmelidir.
2. **IPS Simulation Mode Doğrulaması:**
   - IPS yönetim konsolundan ilgili Source ve Target sistem seçilerek **Run Simulation** tetiklenir [^20].
   - Üretilen simülasyon raporunda mock kullanıcının hedef sistemde yaratılacağı (`CREATE`), hangi rolleri alacağı (`ASSIGN`) ve sistemin hata üretip üretmediği doğrulanır.

---

## 4. Rol, Rol Grubu ve Unvan Değişimlerinde Kayıt Defteri ve Revizyon Takibi

Kurumsal denetim (SOX, KVKK, ISO/IEC 27001) gereksinimleri uyarınca; bir kullanıcının yetkisinin ne zaman, kim tarafından, hangi unvan veya gerekçeyle değiştirildiğinin geriye dönük sorgulanabilir olması gerekir [^4][^25].

### 4.1. SAP S/4HANA Tarafında Yetki Değişim Kayıt Defteri (Change Logs & Tables)

S/4HANA sistemi üzerinde gerçekleşen tüm yetki atamaları ilişkisel tablolarda ve değişiklik belgelerinde (Change Documents) saklanır [^21].

```
+-------------------------------------------------------------------------+
|                  SAP DEĞİŞİKLİK BELGELERİ MİMARİSİ                      |
|                                                                         |
|   SU01 / PFCG / SCIM API  ──►  CDHDR (Değişiklik Başlık Tablosu)        |
|                                 ├── Object Class: USER / IDENTITY       |
|                                 ├── Object ID   : AHMET.YILMAZ          |
|                                 ├── Change No   : 0001248590            |
|                                 ├── User Name   : IPS_PROV_SERVICE      |
|                                 └── Timestamp   : 2026-10-27 14:32:10   |
|                                                                         |
|                                CDPOS (Değişiklik Pozisyon Tablosu)      |
|                                 ├── Table Name  : AGR_USERS             |
|                                 ├── Field Name  : AGR_NAME              |
|                                 ├── Change Type : I (Insert) / D (Delete|
|                                 ├── Old Value   : Z:S4_MM_D_BUYER_1000  |
|                                 └── New Value   : Z:S4_FI_S_AP_CLERK    |
+-------------------------------------------------------------------------+
```

#### Kritik Veritabanı Tabloları

1. **`CDHDR` (Change Document Header - Değişiklik Başlığı):** [^21]
   - Değişikliği yapan kullanıcıyı (`USERNAME` - IPS servis kullanıcısı veya Basis uzmanı), işlem tarihini (`UDATE`), saatini (`UTIME`) ve değişiklik numarasını (`CHANGENR`) tutar.
   - Nesne Sınıfı (`OBJECTCLAS`): `USER` veya `IDENTITY`.

2. **`CDPOS` (Change Document Position - Değişiklik Detayı):** [^21]
   - Hangi tablonun (`TABNAME`: `AGR_USERS`), hangi alanının (`FNAME`: `AGR_NAME`) değiştiğini; eski değer (`VALUE_OLD`) ve yeni değer (`VALUE_NEW`) kırılımıyla kaydeder.
   - Değişiklik Tipi (`CHNGIND`):
     - `I` : Insert (Yeni rol eklendi).
     - `D` : Delete (Rol geri alındı/silindi).
     - `U` : Update (Geçerlilik tarihi güncellendi).

3. **`USH02` (Logon Data Change History):** [^21]
   - Kullanıcı kilitlenme durumlarını (`UFLAG`), şifre değişikliklerini ve hesap aktiflik geçmişini izler.

4. **`USH04` (Authorization Profile History):** [^21]
   - Kullanıcıya bağlanan profil ve yetki değişikliklerinin tarihsel kopyasını tutar.

5. **`AGR_USERS` (User Assignments to Roles):** [^11]
   - Kullanıcının aktif ve geçmiş tüm rol bağlantılarını `FROM_DAT` ve `TO_DAT` tarihleriyle tutar.

#### SUIM (User Information System) Denetim Raporları

- **`RSUSR100N` İşlem Kodu (Change Documents for Users):** [^22]
  Kullanıcı bazında rol ekleme, silme, hesap kilitleme ve profil değişikliklerinin raporlandığı standart denetim arayüzüdür.
- **`RSUSR100U` İşlem Kodu (Change Documents for Roles):** [^22]
  PFCG rolünün kendi içindeki yetki nesnesi, yetki alanı veya menü değişikliklerini takip eden rapordur.

---

### 4.2. SAP IPS Senkronizasyon Kayıt Defteri (Provisioning Logs)

IPS motorunun gerçekleştirdiği her işlem yönetim arayüzünde kayıt altına alınır [^23]:

1. **Execution Details Raporu:**
   - Her senkronizasyon çalıştırmasında (Job Run) üretilen özet istatistikler:
     - `Read Entities` : Kaynaktan okunan kayıt sayısı.
     - `Created Entities` : Hedefte yeni açılan kayıt sayısı.
     - `Updated Entities` : Hedefte güncellenen kayıt sayısı.
     - `Deleted Entities` : Hedefte silinen veya kilitlenen kayıt sayısı.
     - `Failed Entities` : Hata alan kayıt sayısı.
2. **Detailed Execution Log (CSV / JSON Dökümü):**
   - Başarısız olan veya atlanan işlemlerin hata kodları (`HTTP 409 Conflict`, `HTTP 400 Bad Request`, `HTTP 404 Not Found`).
   - Bu loglar kurumsal SIEM (Splunk, Microsoft Sentinel) sistemlerine aktarılarak merkezi denetim sağlanabilir [^23].

---

### 4.3. Microsoft Entra ID Denetim Günlükleri (Audit Logs)

Rol değişiminin ilk tetiklendiği yer olan kurumsal dizindeki kayıtlar [^24]:

- **Kategori:** `Directory Management` $\rightarrow$ `Group Management`.
- **Aktivite:**
  - `Add member to group` (Kullanıcı `SG_SAP_*` grubuna eklendi).
  - `Remove member from group` (Kullanıcı `SG_SAP_*` grubundan çıkarıldı).
- **Aktör Bilgisi (Initiated By):**
  - SuccessFactors inbound sync servis hesabı veya İK yetkilisi.

---

### 4.4. Noktasal Zamanlı Denetim (Point-in-Time Audit) ve Revizyon Modeli

Denetçiler genellikle şu soruyu sorar: *"Ahmet Bey 15 Mart 2026 tarihinde hangi SAP yetkilerine sahipti ve bu yetki hangi unvan gerekçesiyle verilmişti?"* Bu sorgulama SOX 404 ve ISO 27001 gereğidir [^25].

Bu sorunun cevabını üretmek için 3 katmanlı revizyon korelasyonu kullanılır:
1. **İK Pozisyon Kaydı:** SF / İK sistemindeki pozisyon geçmişi tablosu (15 Mart'taki unvanı).
2. **Dizin Grup Üyeliği:** Entra ID Audit Log / Reporting üzerinden 15 Mart'taki grup üyelikleri [^24].
3. **S/4HANA Yetki Kaydı:** `AGR_USERS` tablosunda `FROM_DAT <= '20260315' AND TO_DAT >= '20260315'` koşulunu sağlayan roller [^11][^21].

---

## 5. Rol Grubu veya Unvanın Pasifleştirilmesi ve Emekli Edilmesi (Lifecycle Management)

Kurumsal organizasyonlar dinamiktir; zamanla unvanlar birleşir, modül rolleri yenilenir veya eski süreçler yürürlükten kalkar. Bu durumlarda rolün veya unvanın kontrollü şekilde sistemden emekli edilmesi gerekir.

```
+-------------------------------------------------------------------------+
|                  ROL VE UNVAN EMEKLİLİK YAŞAM DÖNGÜSÜ                    |
|                                                                         |
|   1. DUYURU & İŞARETLEME  ──► [DEPRECATED] Etiketi ve Yeni Atamaya Kapat |
|                                                                         |
|   2. GEÇERLİLİK BİTİRME   ──► AGR_USERS TO_DAT = SY-DATUM (Toplu Çıkış) |
|                                                                         |
|   3. AD GRUP TASFİYESİ    ──► SG_SAP_RETIRED_* Yeniden Adlandırma       |
|                                                                         |
|   4. KARANTİNA DÖNEMİ     ──► 90 Gün Pasif Bekletme (Geri Dönüş Payı)   |
|                                                                         |
|   5. KALICI SİLME         ──► Rolün PFCG ve AD'den Tamamen Kaldırılması |
+-------------------------------------------------------------------------+
```

### 5.1. Kavramsal Ayrım: Pasifleştirme vs. Emekli Etme

- **Pasifleştirme (Deactivation - Geçici Dondurma):**
  - Rolün veya unvanın yeni personele atanması durdurulur.
  - Mevcut kullanıcıların yetkisi korunur veya geçici bir süre için askıya alınır.
  - Geri dönüşü kolaydır; bayrak veya tarih güncellenerek yeniden açılabilir.
- **Emekli Etme (Deprecation / Retirement - Kalıcı Tasfiye):**
  - Rolün veya unvanın kurumsal geçerliliği kalıcı olarak sonlandırılır.
  - Rolü kullanan tüm mevcut personeller alternatif yeni rollere göç ettirilir.
  - Belirlenen karantina süresi sonunda teknik rol sistemden tamamen silinir.

---

### 5.2. SAP S/4HANA Tarafında Kademeli Rol Emeklilik Prosedürü

1. **Aşama 1: Rol Açıklamasının Güncellenmesi ve Kilitlenmesi:**
   - PFCG işlem kodunda rolün tanımına `[DEPRECATED - KULLANILMAZ]` ibaresi eklenir.
   - Rol menüsü ve yetkileri dondurulur; yetki ekibi tarafından yeni kullanıcı atanması yasaklanır.
2. **Aşama 2: Toplu Geçerlilik Bitiş Tarihi Güncellemesi (`Mass Role Expiration`):**
   - Rolü kullanan aktif hesapların `AGR_USERS` tablosundaki bitiş tarihi bugünün tarihine çekilir:
     `TO_DAT = SY-DATUM`.
   - Bu işlem SAP standart raporu `PRGN_COMPRESS_TIMES` veya özel Basis kütüğü ile toplu olarak çalıştırılır [^26].
   - Kullanıcıların oturumu ertesi gün açıldığında ilgili rol otomatik olarak devre dışı kalır.
3. **Aşama 3: Karantina Süresi (Quarantine Period):**
   - Rol sistemden hemen silinmez. Asgari **90 gün** boyunca pasif halde bekletilir.
   - Bu süre zarfında iş süreçlerinde beklenmeyen bir aksama yaşanırsa tarih uzatılarak hızlı geri dönüş sağlanabilir.
4. **Aşama 4: Kalıcı Tasfiye ve Arşivleme:**
   - Karantina süresi dolduğunda, rolün yetki dökümü PDF/metin olarak arşivlenir.
   - Rol PFCG üzerinden silinir ve değişiklik belgesi denetim için saklanır [^21].

---

### 5.3. Microsoft Entra ID ve IPS Tarafında Tasfiye Prosedürü

1. **Grup İsimlendirmesinin Değiştirilmesi:**
   - Emekli edilen güvenlik grubu silinmeden önce yeniden adlandırılır:
     `SG_SAP_S4P_MM_BUYER_1000` $\rightarrow$ `SG_SAP_RETIRED_MM_BUYER_1000`.
2. **IPS Kaynak Filtresi ile Otomasyon Dışına Alma:**
   - IPS Source dönüşüm filtresine retired öneki eklenir [^27]:
     `displayName startswith 'SG_SAP_' and not (displayName startswith 'SG_SAP_RETIRED_')`
   - Bu sayede IPS artık bu grubu okumaz ve hedef sisteme aktarmaz.
3. **Yetim Yetki (Orphan Entitlement) Temizliği:**
   - Hedef sistemde `authoritative = false` ayarlı olduğu için, grup IPS kapsamından çıktığında kullanıcıların S/4HANA üzerindeki eski rolleri otomatik silinmeyebilir [^17].
   - Bu yetim rollerin temizlenmesi için iki yöntem uygulanır:
     - **Yöntem A (Planlı Göç):** Kullanıcılar AD tarafında yeni role (`SG_SAP_S4P_MM_BUYER_V2`) taşınırken eski rolün bitiş tarihi `AGR_USERS` üzerinden kapatılır.
     - **Yöntem B (Kontrollü Temizlik Raporu):** Basis ekibi `SUIM` üzerinden emekli edilen role sahip kullanıcıları listeleyerek tek seferlik toplu silme işlemi uygular [^22].

---

### 5.4. Unvan (Job Title / Position) Emeklilik ve Geçiş Matrisi

İnsan Kaynakları tarafında bir unvan veya pozisyon kapatıldığında (Örn: "Satınalma Şefi" unvanının "Kategori Yöneticisi" olarak güncellenmesi):

1. **SF Pozisyon Kapatma:** SuccessFactors üzerinde eski pozisyon kodu pasife alınır.
2. **AD Senkronizasyonu:** IT dizin entegrasyonu kullanıcının unvan ve departman alanını günceller.
3. **Grup Değişimi:** Kullanıcı eski pozisyonun grubundan çıkarılır, yeni pozisyonun grubuna eklenir.
4. **IPS Otomatik Eşitlemesi:**
   - IPS bir sonraki çalıştırmada kullanıcının eski S/4HANA rolünü geri alır (`Operation: remove`) [^3].
   - Yeni unvanın gerektirdiği rolleri kullanıcıya tanımlar (`Operation: add`) [^3].
   - Bu akış sayesinde kullanıcı yeni görevine başladığı gün yetki karmaşası yaşamadan güncel yetkileriyle çalışmaya devam eder.

---

## 6. Özet Karar ve Araştırma Matrisi

| Konu | Temel Teknik Bulgu / İlke | Alınması Gereken Önlem / Standart |
|:---|:---|:---|
| **Rol İsimlendirmesi** | SAP'de teknik rol ismi azami 30 karakterdir (`AGR_NAME`) [^5]. | `Z:<KAPSAM>_<MODÜL>_<TİP>_<İŞLEV>_<ORGANİZASYON>` standardı bağlayıcı kural olarak belirlenmelidir [^6]. |
| **Varsayılan Atamalar** | SCIM doğrudan SU3 parametresi yazamaz [^13]; geçerlilik varsayılanı `31.12.9999`'dur [^11]. | Veri kısıtları kullanıcı parametresiyle değil, türetilmiş rollerle (Derived Roles) çözülmelidir [^7]. |
| **Çift Yönlü Aktarım** | Karşılıklı kullanıcı yazma döngüsel çakışmaya (Circular Loop) yol açar [^16]. | Entra ID tekil kimlik kaynağı (Source) kalmalı; S/4HANA yalnızca hedef (Target) olmalıdır [^17]. |
| **Kayıt Defteri & Revizyon** | Rol değişimleri `CDHDR` ve `CDPOS` tablolarında; kullanıcı geçmişi `USH02/USH04`'tedir [^21]. | Denetim için `RSUSR100N` [^22] ve IPS execution logları [^23] merkezi SIEM sistemine yönlendirilmelidir. |
| **Emeklilik Yaşam Döngüsü** | Kullanılmayan roller doğrudan silinirse iş sürekliliği aksayabilir. | 90 günlük karantina süresi, `[DEPRECATED]` etiketi [^26] ve `SG_SAP_RETIRED_*` filtresi [^27] işletilmelidir. |

---

## 7. Resmi Referanslar ve Kaynakça (Official References & Standards)

[^1]: **SAP Help Portal - SAP NetWeaver AS ABAP Security Guide:** *Role and User Concept in AS ABAP*. [https://help.sap.com/docs/ABAP_PLATFORM_NW756/e37d57a2707d4c829c36209b5523dd06/488e3ad7bb0c4415e10000000a42189b.html](https://help.sap.com/docs/ABAP_PLATFORM_NW756/e37d57a2707d4c829c36209b5523dd06/488e3ad7bb0c4415e10000000a42189b.html)
[^2]: **SAP Help Portal - SAP S/4HANA Authorizations Concept:** *User and Role Maintenance in Transaction PFCG*. [https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE)
[^3]: **IETF SCIM 2.0 Specifications:** *RFC 7643 - System for Cross-domain Identity Management: Core Schema* & *RFC 7644 - Protocol Specification (Section 3.5.2 Modifying with PATCH)*. [https://datatracker.ietf.org/doc/html/rfc7643](https://datatracker.ietf.org/doc/html/rfc7643) / [https://datatracker.ietf.org/doc/html/rfc7644](https://datatracker.ietf.org/doc/html/rfc7644)
[^4]: **U.S. National Institute of Standards and Technology & ISO:** *NIST SP 800-207: Zero Trust Architecture* & *ISO/IEC 27001:2022 Information Security - Controls 5.18 (Access Rights) & 8.15 (Logging)*. [https://doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207) / [https://www.iso.org/standard/27001](https://www.iso.org/standard/27001)
[^5]: **SAP Note 1581630 & Data Dictionary:** *Length Limitation of Role Names in PFCG (Table AGR_DEFINE - Field AGR_NAME maximum 30 characters)*. [https://me.sap.com/notes/1581630](https://me.sap.com/notes/1581630)
[^6]: **SAP Note 164894:** *Customer Name Range for Roles, Authorizations, and Authorization Objects (Namespace Z* and Y*)*. [https://me.sap.com/notes/164894](https://me.sap.com/notes/164894)
[^7]: **SAP Note 323817 & SAP Help Portal:** *Derived Roles Concept - Inheritance of Menus and Maintenance of Organizational Levels (`$BUKRS`, `$WERKS`)*. [https://me.sap.com/notes/323817](https://me.sap.com/notes/323817)
[^8]: **SAP Note 2887640 & SAP Fiori Guide:** *Managing Launchpad Spaces and Pages in SAP S/4HANA via Fiori Launchpad Content Manager (`/UI2/FLPCM_CUST`)*. [https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE)
[^9]: **SAP Help Portal - SAP BTP Security:** *Role Collections, Application Roles, and Scopes in Cloud Foundry and Kyma Environments*. [https://help.sap.com/docs/btp/sap-business-technology-platform/security](https://help.sap.com/docs/btp/sap-business-technology-platform/security)
[^10]: **Microsoft Learn - Microsoft Entra ID:** *Manage Security Groups, Access Reviews, and Graph API Directory Optimization*. [https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/)
[^11]: **SAP Note 1735919 & SAP NetWeaver Data Dictionary:** *User Master Record and Role Assignment Validity Constraints in Table AGR_USERS (`FROM_DAT`, `TO_DAT = 31.12.9999`)*. [https://me.sap.com/notes/1735919](https://me.sap.com/notes/1735919)
[^12]: **SAP Help Portal - AS ABAP User Maintenance:** *User Types in SAP Systems: Dialog (A), System (B), Communication (C), Service (S), Reference (L)*. [https://help.sap.com/docs/SAP_NETWEAVER_750](https://help.sap.com/docs/SAP_NETWEAVER_750)
[^13]: **SAP Note 2530188:** *SCIM 2.0 Integration Boundaries - Absence of Native SU3 / USR05 User Parameters in SCIM Core Schema*. [https://me.sap.com/notes/2530188](https://me.sap.com/notes/2530188)
[^14]: **SAP Help Portal - Identity Provisioning (IPS):** *Transformation Syntax - Using `defaultValue`, `condition`, and `optional` attributes in Provisioning Systems*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/transformation-expressions](https://help.sap.com/docs/identity-provisioning/identity-provisioning/transformation-expressions)
[^15]: **SAP Help Portal - SAP BTP Subaccount Administration:** *Trust and Federation Configuration - Assigning Default Role Collections to Authenticated Users*. [https://help.sap.com/docs/btp/sap-business-technology-platform/trust-and-federation](https://help.sap.com/docs/btp/sap-business-technology-platform/trust-and-federation)
[^16]: **IETF RFC 7644 - Section 3.14:** *SCIM Protocol Concurrency and Loop Detection in Multi-System Topologies*. [https://datatracker.ietf.org/doc/html/rfc7644#section-3.14](https://datatracker.ietf.org/doc/html/rfc7644#section-3.14)
[^17]: **SAP Help Portal - Identity Provisioning:** *Target System Properties - Preserving Local Roles with `s4hana.authoritative.role.management = false`*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties](https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties)
[^18]: **SAP Help Portal - SAP Access Control (GRC):** *Emergency Access Management (EAM) - Firefighter Role Protection and Exception Filters*. [https://help.sap.com/docs/SAP_ACCESS_CONTROL](https://help.sap.com/docs/SAP_ACCESS_CONTROL)
[^19]: **SAP Business Accelerator Hub:** *SAP Cloud Identity Services Identity Directory SCIM 2.0 Schema Specification (`urn:ietf:params:scim:schemas:extension:sap:2.0:User`)*. [https://api.sap.com/api/IdentityDirectory/overview](https://api.sap.com/api/IdentityDirectory/overview)
[^20]: **SAP Help Portal - Identity Provisioning:** *Simulate Provisioning Jobs and Pre-Validate User Transformations*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/simulate-provisioning-jobs](https://help.sap.com/docs/identity-provisioning/identity-provisioning/simulate-provisioning-jobs)
[^21]: **SAP Help Portal - SAP ABAP Change Documents Architecture:** *Tracking User Master and Authorization Profile Modifications via Tables CDHDR, CDPOS, USH02, and USH04*. [https://help.sap.com/docs/SAP_NETWEAVER_750](https://help.sap.com/docs/SAP_NETWEAVER_750)
[^22]: **SAP Note 164286:** *Evaluating Change Documents for Users and Roles in SUIM (Transactions RSUSR100N and RSUSR100U)*. [https://me.sap.com/notes/164286](https://me.sap.com/notes/164286)
[^23]: **SAP Help Portal - Identity Provisioning:** *Managing Job Execution Logs and Streaming Provisioning Events to Enterprise SIEM (Splunk, Microsoft Sentinel)*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/manage-provisioning-logs](https://help.sap.com/docs/identity-provisioning/identity-provisioning/manage-provisioning-logs)
[^24]: **Microsoft Learn - Microsoft Entra Monitoring & Health:** *Audit Logs in Microsoft Entra ID - Group Lifecycle and Member Activity Audit Trail*. [https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-logs)
[^25]: **U.S. Securities and Exchange Commission (SEC):** *Sarbanes-Oxley Act of 2002 Section 404 - Management Assessment of Internal Controls Over Financial Reporting and ITGC Point-in-Time User Access Reviews*. [https://www.sec.gov/rules/final/33-8238.htm](https://www.sec.gov/rules/final/33-8238.htm)
[^26]: **SAP Note 313947 & ABAP Report PRGN_COMPRESS_TIMES:** *Automatic Adjustment and Expiration of Role Assignments in Table AGR_USERS*. [https://me.sap.com/notes/313947](https://me.sap.com/notes/313947)
[^27]: **SAP Help Portal - Identity Provisioning:** *Filtering Entities in Source System Transformations using SCIM Filtering Strings*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/filter-entities](https://help.sap.com/docs/identity-provisioning/identity-provisioning/filter-entities)

---

## 8. Kısaltmalar ve Terimler Sözlüğü (Glossary of Acronyms & Abbreviations)

Belge boyunca geçen tüm teknik kısaltmalar, protokoller ve kavramlar alfabetik olarak aşağıda tanımlanmıştır:

| Kısaltma | Orijinal Açılımı (İngilizce / Resmi) | Belgedeki Teknik Anlamı ve İşlevi |
|:---|:---|:---|
| **ABAC** | Attribute-Based Access Control | Nitelik Tabanlı Erişim Denetimi. Kullanıcının unvan, departman, şirket kodu gibi anlık profil niteliklerine göre dinamik veri kısıtlaması (BTP CAP CDS katmanı). |
| **ABAP** | Advanced Business Application Programming | SAP'nin kurumsal ERP iş mantığını koşturan ve yetki denetimlerini (`AUTHORITY-CHECK`) yürüten çekirdek programlama dili. |
| **AD** | Active Directory | Şirket içi yerel ağlarda kullanıcı, bilgisayar ve güvenlik gruplarını yöneten Microsoft dizin hizmeti. |
| **API** | Application Programming Interface | Uygulama Programlama Arayüzü. Sistemlerin programatik olarak veri alışverişi yaptığı bağlantı noktası. |
| **BC** | Basis Components | SAP Basis. SAP sistem altyapısını, veritabanını, sistem parametrelerini ve kullanıcı güvenliğini yöneten temel sistem yönetim katmanı ve modülü. |
| **BDDK** | Bankacılık Düzenleme ve Denetleme Kurumu | Türkiye finans sektöründe bilgi sistemleri güvenliği, yetki yönetişimi ve denetim izi standartlarını belirleyen yasal düzenleyici otorite. |
| **BTP** | SAP Business Technology Platform | SAP'nin bulut entegrasyon, uzantı geliştirme, kimlik federasyonu ve merkezi yetkilendirme platformu. |
| **BW / BW/4HANA** | Business Warehouse / Business Warehouse 4 HANA | SAP'nin kurumsal veri ambarı, analitik modelleme ve raporlama sistemi. |
| **CAP** | Cloud Application Programming Model | SAP BTP üzerinde Java ve Node.js ile kurumsal mikroservisler geliştirmek için kullanılan resmi mimari çerçeve. |
| **CDHDR** | Change Document Header | SAP veritabanında kullanıcı, rol ve profil değişikliklerinin başlık bilgilerini (`USERNAME`, `UDATE`, `UTIME`, `CHANGENR`) saklayan sistem tablosu. |
| **CDPOS** | Change Document Position | SAP veritabanında hangi tablonun hangi alanının değiştiğini (`TABNAME`, `FNAME`, `VALUE_OLD`, `VALUE_NEW`, `CHNGIND`) tutan değişiklik kalem tablosu. |
| **CDS** | Core Data Services | SAP HANA ve BTP CAP üzerinde veri modellerini ve deklaratif erişim kurallarını (`@restrict`) tanımlayan şema dili. |
| **CISO** | Chief Information Security Officer | Kurumsal Bilgi Güvenliği Baş Yetkilisi. Yetki politikalarını, acil durum kasa (Break-Glass) onaylarını ve denetim stratejilerini belirleyen üst yönetici. |
| **CO** | Controlling | SAP Maliyet Muhasebesi ve Kontrol Modülü. Masraf merkezleri ve kâr merkezleri bazında yetki kısıtlamalarını yöneten fonksiyonel modül. |
| **CSV** | Comma-Separated Values | Virgülle ayrılmış metin veri formatı. IPS simülasyon ve senkronizasyon raporlarının dışa aktarılmasında kullanılır. |
| **CUA** | Central User Administration | Merkezi Kullanıcı Yönetimi. Klasik SAP sistemlerinde kullanıcı ve yetkileri tek bir merkezi sistemden dağıtan geleneksel RFC tabanlı teknoloji. |
| **EAM** | Emergency Access Management | Acil Erişim Yönetimi. SAP GRC modülünde süper yönetici yetkilerinin süreli, kayıtlı ve onaylı olarak verilmesini sağlayan yangın söndürücü (Firefighter) mekanizması. |
| **ESS** | Employee Self-Service | Çalışan Öz-Servisi. Tüm şirket personelinin izin talebi, bordro görüntüleme gibi temel işlemleri yaptığı Fiori portal arayüzü (`SG_SAP_ALL_EMPLOYEES`). |
| **FI** | Financial Accounting | SAP Finansal Muhasebe Modülü. Genel muhasebe, borçlar/alacaklar muhasebesi ve şirket kodu kısıtlı rolleri barındıran çekirdek ERP modülü. |
| **FUE** | Full User Equivalent | Tam Kullanıcı Eşdeğeri. SAP bulut lisanslamasında farklı kullanıcı tiplerini tek bir metrik havuzunda ağırlıklandıran lisans birimi. |
| **GDPR** | General Data Protection Regulation | Avrupa Birliği Genel Veri Koruma Tüzüğü. Kimlik verilerinin işlenmesi, asgarileştirilmesi ve aktarılmasında uyulması gereken yasal çerçeve. |
| **GRC** | Governance, Risk, and Compliance | Kurumsal Yönetişim, Risk ve Uyumluluk. Görevler Ayrılığı (SoD) çatışmalarını ve acil durum erişimlerini yöneten SAP yazılım paketi. |
| **HCM** | Human Capital Management | SAP İnsan Sermayesi Yönetimi. Personel ana verisi, organizasyonel hiyerarşi ve bordro süreçlerini yöneten sistem. |
| **HR** | Human Resources | İnsan Kaynakları. Kimlik yaşam döngüsünün tetikleyicisi olan kurumsal birim. |
| **HSM** | Hardware Security Module | Donanımsal Güvenlik Modülü. Kriptografik anahtarları, SSL/TLS sertifikalarını ve acil durum şifrelerini fiziksel olarak koruyan yüksek güvenlikli kasa cihazı. |
| **HTTP / HTTPS** | Hypertext Transfer Protocol (Secure) | Web üzerinde güvenli veri iletimi sağlayan çekirdek internet iletişim protokolü. |
| **IAM** | Identity and Access Management | Kimlik ve Erişim Yönetimi. Doğru kişilerin doğru kaynaklara doğru zamanda ve doğru nedenlerle erişmesini sağlayan kurumsal disiplin. |
| **IAS** | SAP Cloud Identity Services - Identity Authentication | SAP bulut ekosisteminin kurumsal kimlik doğrulama, tekil oturum açma (SSO) ve OIDC/SAML federasyon proxy servisi. |
| **IdP** | Identity Provider | Kimlik Sağlayıcı. Kullanıcıların kimliğini doğrulayan ve doğrulanmış dijital biletler (token) üreten merkezi güven makamı (Örn: Microsoft Entra ID). |
| **IETF** | Internet Engineering Task Force | İnternet Mühendisliği Görev Gücü. SCIM, HTTP, TLS gibi küresel internet protokol ve standartlarını yayımlayan uluslararası kuruluş. |
| **IGA** | Identity Governance and Administration | Kimlik Yönetişimi ve İdare. Rol yaşam döngüsü, yetki sertifikasyonu ve erişim taleplerini yöneten üst seviye kurumsal disiplin. |
| **IPS** | SAP Cloud Identity Services - Identity Provisioning | Kimlik ve yetkileri kaynak sistemlerden (Entra ID, SF) okuyup hedef sistemlere (S/4HANA, BTP) SCIM protokolüyle dağıtan bulut ETL provizyon motoru. |
| **ISO** | International Organization for Standardization | Uluslararası Standardizasyon Kuruluşu. Bilgi güvenliği yönetim sistemi standartlarını (ISO/IEC 27001) yayımlayan küresel otorite. |
| **IT** | Information Technology | Bilgi Teknolojileri. Sistem altyapısı, dizin hizmetleri ve yetkilendirme süreçlerini işleten teknik birim. |
| **JML** | Joiner, Mover, Leaver | İşe Giriş (Joiner), Görev Değişimi (Mover) ve İşten Ayrılış (Leaver) adımlarını kapsayan uçtan uca çalışan kimlik yaşam döngüsü modeli. |
| **JSON** | JavaScript Object Notation | SCIM REST API istek ve yanıtlarında kullanılan, insan tarafından okunabilir hafif veri değişim formatı. |
| **KBA** | Knowledge Base Article | SAP Destek Bilgi Bankası Makalesi. SAP ürünlerindeki teknik konfigürasyon, hata giderme ve kılavuzları içeren resmi doküman tipi. |
| **KVKK** | Kişisel Verilerin Korunması Kanunu | 6698 Sayılı Kişisel Verilerin Korunması Kanunu. Türkiye'de kimlik verilerinin işlenmesi ve saklanmasını denetleyen mevzuat. |
| **MFA** | Multi-Factor Authentication | Çok Faktörlü Doğrulama. Kullanıcının oturum açarken parola haricinde ek bir faktör (Authenticator, SMS, FIDO2) ile kimliğini kanıtlamasını zorunlu kılan güvenlik mekanizması. |
| **MM** | Materials Management | SAP Malzeme Yönetimi ve Satınalma Modülü. Satınalma siparişleri, onay süreçleri ve stok yönetimini içeren çekirdek ERP modülü. |
| **NIST** | National Institute of Standards and Technology | ABD Ulusal Standartlar ve Teknoloji Enstitüsü. Sıfır Güven (Zero Trust - SP 800-207) mimarisinin küresel referans otoritesi. |
| **OData** | Open Data Protocol | REST tabanlı web servisleri üzerinden SAP verilerine erişim ve CRUD operasyonları sağlayan açık veri protokolü. |
| **OIDC** | OpenID Connect | OAuth 2.0 üzerine inşa edilmiş, JSON Web Token (JWT) tabanlı modern bulut kimlik doğrulama protokolü. |
| **PAM** | Privileged Access Management | Ayrıcalıklı Erişim Yönetimi. Sistem yöneticisi ve kritik yetkilerin süreli, denetimli ve kasadan çekilerek verilmesini yöneten güvenlik disiplini. |
| **PCE** | Private Cloud Edition | SAP S/4HANA Cloud Özel Bulut Sürümü. Müşteriye özel ayrılmış bulut altyapısında çalışan S/4HANA ERP sistemi. |
| **PFCG** | Profile Generator | SAP ABAP sisteminde yetki rollerinin, işlem kodlarının ve yetki nesnelerinin tasarlandığı temel yönetim işlem kodu. |
| **PM** | Plant Maintenance | SAP Bakım ve Onarım Modülü. Tesis ekipmanları ve bakım iş emirlerini yöneten modül. |
| **PP** | Production Planning | SAP Üretim Planlama Modülü. Üretim emirleri ve kapasite planlama yetkilerini barındıran modül. |
| **QA** | Quality Assurance | Kalite ve Test Ortamı. Yetki değişikliklerinin ve mock simülasyonların canlıya geçmeden önce doğrulandığı sistem katmanı. |
| **RACI** | Responsible, Accountable, Consulted, Informed | Sorumlu, Hesap Verebilir, Danışılan, Bilgilendirilen. Rol ve yetki süreçlerindeki kurumsal görev paylaşım matrisi. |
| **RBAC** | Role-Based Access Control | Rol Tabanlı Erişim Denetimi. Yetkilerin doğrudan kullanıcılara değil, iş fonksiyonlarını temsil eden rollere atanması güvenlik mimarisi. |
| **RFC** | Request for Comments / Remote Function Call | 1. IETF internet standart belgesi (Örn: RFC 7643).<br>2. SAP sistemleri arasında veri ve fonksiyon çağrısı sağlayan ikili iletişim protokolü. |
| **S/4HANA** | SAP Business Suite 4 SAP HANA | SAP'nin yüksek hızlı bellek içi (in-memory) HANA veritabanı üzerinde koşan yeni nesil ERP kurumsal yazılım paketi. |
| **SAC** | SAP Analytics Cloud | SAP'nin BTP üzerinde çalışan bulut tabanlı iş zekası, analitik ve planlama platformu. |
| **SAML** | Security Assertion Markup Language | XML tabanlı, kurumlar arası tekil oturum açma (SSO) ve yetki federasyonu sağlayan açık güvenlik standardı (SAML 2.0). |
| **SCIM** | System for Cross-domain Identity Management | IETF RFC 7643/7644 standartlarına dayalı, REST/JSON tabanlı bulut kimlik provizyonu ve şema yönetim protokolü. |
| **SD** | Sales and Distribution | SAP Satış ve Dağıtım Modülü. Müşteri siparişleri, sevkiyat ve faturalama yetkilerini içeren çekirdek modül. |
| **SF** | SAP SuccessFactors | Çalışan ana verisinin, pozisyon kodlarının ve istihdam hareketlerinin tutulduğu tekil güven kaynağı (SSoT) bulut İK sistemi. |
| **SIEM** | Security Information and Event Management | Güvenlik Bilgi ve Olay Yönetimi. IPS ve S/4HANA denetim loglarının toplandığı kurumsal izleme platformu (Splunk, Microsoft Sentinel). |
| **SoD** | Segregation of Duties | Görevler Ayrılığı. Finansal veya operasyonel suistimalleri önlemek amacıyla birbirini denetleyen işlemlerin farklı kullanıcılara verilmesi kuralı. |
| **SOX** | Sarbanes-Oxley Act of 2002 | ABD Şirketler Muhasebe Reformu Kanunu. Bölüm 404 uyarınca IT genel kontrolleri (ITGC) ve noktasal zamanlı yetki denetimlerini zorunlu kılan yasal düzenleme. |
| **SSoT** | Single Source of Truth | Tekil Güven Kaynağı. Kurumsal kimlik veya yetki verisinin yalnızca tek bir ana sistemden doğması ve diğer sistemlerin bu kaynağı esas alması ilkesi. |
| **SUIM** | User Information System | SAP Kullanıcı Bilgi Sistemi. Kullanıcı, rol, profil ve yetki değişiklik belgelerinin geriye dönük sorgulandığı denetim işlem kodları bütünü (`RSUSR100N`, `RSUSR100U`). |
| **T-Code** | Transaction Code | SAP İşlem Kodu. SAP GUI veya arayüzünde belirli bir uygulama ekranını veya fonksiyonu doğrudan başlatan kısa komut kodu (Örn: `PFCG`, `SU01`, `SUIM`). |
| **UI** | User Interface | Kullanıcı Arayüzü. Kullanıcıların sistemle etkileşime girdiği görsel katman (SAP GUI, Fiori Launchpad). |
| **UPN** | User Principal Name | Active Directory ve Microsoft Entra ID'de kullanıcının e-posta biçimindeki benzersiz oturum açma adı (Örn: `ahmet.yilmaz@corp.com`). |
| **UUID** | Universally Unique Identifier | Evrensel Benzersiz Kimlik. Sistemler arası eşleştirmede kullanılan 128-bitlik benzersiz kimlik dizgisi (Örn: `a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d`). |
| **XSUAA** | Extended Services User Account and Authentication | SAP BTP Cloud Foundry ortamında OAuth 2.0 token üreten, Role Collection ve scope atamalarını yöneten çekirdek yetkilendirme servisi. |
