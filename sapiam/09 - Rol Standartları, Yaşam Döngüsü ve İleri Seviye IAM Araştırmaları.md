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
  - connectivity
  - principal-propagation
  - ai-governance
  - cds-dcl
  - user-buffer
status: approved
version: "3.2"
updated: 2026-11-03
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Rol İsimlendirme Standartları, Yaşam Döngüsü ve İleri Seviye IAM Araştırmaları

Bu doküman; SAP S/4HANA (On-Premise / Private Cloud / Public Cloud), SAP Business Technology Platform (BTP), SAP Cloud Identity Services (IAS/IPS) ve kurumsal kimlik yönetişim platformu (Microsoft Entra ID) entegrasyonunda yetki mimarisini, rol isimlendirme standartlarını, rol tasarım stratejilerini, çift yönlü senkronizasyon modellerini, bağlantı ve Principal Propagation mekanizmalarını, yeni nesil yapay zeka (SAP AI Core & Joule) IAM yönetişimini, CDS DCL veri güvenliğini, denetim kayıt defteri mekanizmalarını ve rol/unvan emeklilik yaşam döngüsü prosedürlerini uçtan uca teknik derinlikte incelemek amacıyla hazırlanmıştır.

İlişkili Dokümanlar:
* İnteraktif Yönetici Sunumu: [[sunum_rol_standartlari.html|Rol Standartları ve Yaşam Döngüsü Sunumu]]
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Kurumsal Strateji ve ROI: [[01 - Kurumsal Strateji, Olgunluk ve ROI]]
* Teknik Keşif Kılavuzu: [[02 - Teknik Keşif Anketi (Workshop Guide)]]
* Çözüm Mimarisi ve JML: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* JSON Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Güvenlik, Denetim & Kasa: [[05 - Güvenlik, Denetim ve Break-Glass]]
* Kurulum ve Doğrulama: [[06 - Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi]]
* Taktiksel Operasyon ve Sınırlar: [[07 - Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi]]
* Terimler ve Kısaltmalar: [[08 - Terimler ve Kısaltmalar Sözlüğü (Glossary)]]

Dokümandaki teknik desenler; **SAP Güvenlik Kılavuzları ve NetWeaver BC-SEC Yetkilendirme Konsepti** [^1][^2][^5], **SAP Destek ve GRC Standartları** [^3], **SAP Activate Metodolojisi** [^7], **Microsoft Entra ID Governance & Güvenlik Kılavuzları** [^8][^9][^10][^11], **SAP AI Core & Generative AI Hub Resmi Spesifikasyonları** [^12], **SAP BTP Connectivity Service Kılavuzu** [^13], **SAP Cloud Identity Services - Identity Authentication & Provisioning** [^10][^14], **SAP BTP Mimari ve Güvenlik Rehberi** [^15], **SAP ABAP Core Data Services (CDS) DCL Güvenlik Kılavuzu** [^16], **SAP Yetki Tampon Belleği (User Buffer) ve PFUD Spesifikasyonu** [^17], **IETF Standartları (SCIM, OAuth, SAML)** [^18][^19][^20][^21] ve küresel denetim regülasyonlarına (SOX 404, NIST SP 800-207, ISO/IEC 27001:2022) [^22][^23][^24] dayandırılmıştır.

---

## 1. SAP Rol Tasarım Stratejisi ve Standartlaştırılmış Rol Adlandırma Kuralları (Naming Conventions)

Yetki yönetiminde kurumsal sürdürülebilirlik, rollerin teknik isminden rolün türünün, etki alanının, modülünün ve veri kısıtının anlaşılabilmesine ve modüler bir mimari şablona oturmasına bağlıdır. Standart ve öngörülebilir bir isimlendirme kuralı olmadan merkezi IAM, IGA veya IPS otomasyonu yürütülemez [^3][^8].

### 1.1. PFCG Teknik Rol İsimlendirme Sözdizimi (Syntax ve Kısıtlar - 30 Karakter Kuralı)

SAP ABAP ve S/4HANA veritabanında teknik rol ismi (`AGR_NAME`) azami **30 karakter** ile sınırlandırılmıştır [^1][^5]. Bu mimari kısıt, isimlendirme standartlarının gereksiz uzatmalardan arındırılarak modüler, hiyerarşik ve sıkıştırılmış kod bloklarıyla tasarlanmasını zorunlu kılar.

```
+-------------------------------------------------------------------------+
|                  SAP PFCG ROL İSİMLENDİRME STANDARDI                    |
|                                                                         |
|   Z _ <KAPSAM> _ <MODÜL> _ <TİP> _ <İŞLEV / UNVAN> _ <ORGANİZASYON>     |
|   ─   ───────   ───────   ─────   ───────────────   ──────────────     |
|  Önek  Sistem    SAP       Rol     Fonksiyonel      Veri Kısıtı        |
|  Alanı Kapsamı   Modülü    Tipi    Görev Tanımı     (Şirket/Tesis)     |
+-------------------------------------------------------------------------+
```

> [!WARNING]
> **ABAP PFCG Rol İsimlendirmesinde İki Nokta Üst Üste (`:`) Riski ve BAPI Kısıtları:**  
> Bazı eski şablonlarda veya dokümantasyonlarda `Z:` şeklinde iki nokta kullanımı görülse de, SAP NetWeaver / S/4HANA ABAP çekirdeğinde rol adlarında `:` karakteri bulunması ciddi mimari riskler doğurur:
> 1. SAP yetki kontrol mekanizmasında (`AUTHORITY-CHECK`) iki nokta (`:`) karakteri özel bir gösterge veya "dummy" yetki nesnesi ayıraç muamelesi görebilir.
> 2. Birçok üçüncü parti Identity Management (IdM) bağlayıcısı (connector), BAPI (`BAPI_USER_ACTGROUPS_ASSIGN`) ve RFC arabirimi iki nokta karakterini bir alan ayrıştırıcısı (delimiter) olarak algılayıp regex hatalarına ve senkronizasyon kesintilerine yol açar.
> 3. Bu nedenle kurumsal SAP standardı, önek olarak kesinlikle iki nokta (`:`) değil; alt çizgi (`Z_` / `Y_`) veya SAP tarafından resmi olarak tescil edilmiş müşteri ad alanı (`/MUSTERI/`) formatını zorunlu kılar [^1][^2].

#### Alan Kodlamaları ve Karakter Dağılımı

1. **Önek Alanı (Namespace - 2 Karakter):**
   - `Z_` : Müşteri içi özelleştirilen ve yerel olarak yönetilen standart müşteri yetki alanı (Önerilen resmi standart) [^1][^2].
   - `Y_` : İkincil müşteri ad alanı (Grup şirketleri, birleşmeler veya global şablon ayrımı için) [^2].
   - `/CORP/` : SAP tescilli kurumsal müşteri ad alanı (Global ölçekli holding yapılarında çakışmaları önlemek için).
   - `SAP_` : SAP standart fabrika teslimi roller. Bu roller güvenlik prensibi gereği son kullanıcılara doğrudan atanmaz; `Z_` alanına kopyalanarak özelleştirilir [^1][^2].

2. **Sistem Kapsamı (2 Karakter):**
   - `S4` : SAP S/4HANA On-Premise veya Private Cloud Edition (PCE) çekirdek ERP sistemi.
   - `BT` : SAP Business Technology Platform (BTP) entegrasyon ve uzantı rolleri.
   - `BW` : SAP BW/4HANA analitik ve kurumsal raporlama yetkileri.
   - `PI` : Process Integration / Process Orchestration teknik rolleri.

3. **Modül Kodu (2 Karakter):**
   - `MM` : Malzeme Yönetimi ve Satınalma (Materials Management).
   - `FI` : Finansal Muhasebe (Financial Accounting).
   - `CO` : Maliyet Muhasebesi ve Kontrol (Controlling).
   - `SD` : Satış ve Dağıtım (Sales and Distribution).
   - `PP` : Üretim Planlama (Production Planning).
   - `PM` : Bakım ve Onarım (Plant Maintenance).
   - `HR` : İnsan Kaynakları ve Bordro (HCM / SuccessFactors entegrasyonu).
   - `BC` : Basis ve Sistem Yönetimi (Basis Components).

4. **Rol Tipi Kodlaması (1 Karakter):**
   - `S` (Single Role - Tekil Fonksiyonel Rol): İşlem kodları (T-Code), yetki nesneleri ve alan değerlerini barındıran temel yetki birimi [^1].
   - `D` (Derived Role - Türetilmiş Rol): Menü ve işlem kodlarını ana (Master) rolden miras alan, sadece organizasyonel seviye değerleri (`$BUKRS`, `$WERKS`, `$EKORG`, `$VKORG`) özelleştirilmiş operasyonel rol [^5].
   - `C` (Composite Role - Birleşik Rol): Menü veya yetki nesnesi içermeyen, kullanıcıya tek seferde birden çok tekil ve türetilmiş rol bağlayan paket rol [^4].
   - `M` (Master / Template Role - Şablon Rol): Kullanıcıya doğrudan atanmayan, menü yapısını ve fonksiyonel yetki nesnelerini barındıran, yalnızca türetilmiş rollere miras veren şablon [^5].

5. **Fonksiyonel İşlev / Görev Tanımı (Azami 10-12 Karakter):**
   - İlgili personelin operasyonel görevini belirten standart kısaltma (Örn: `BUYER_OP`, `GL_ACCOUNT`, `WH_CLERK`, `APPROVER`, `INV_AUDIT`).

6. **Organizasyonel Veri Kısıtı (Azami 4-6 Karakter):**
   - Şirket kodu (`BUKRS`), tesis (`WERKS`) veya organizasyon birimi kodu (Örn: `1000`, `2000`, `GLOBAL`, `ALL`) [^5].

#### Standartlaştırılmış PFCG Rol İsimlendirme Örnekleri

| Teknik Rol İsmi | Rol Türü | Modül | Açıklama |
|:---|:---|:---|:---|
| `Z_S4_MM_M_BUYER_TMPL` | Master (Şablon) | MM | Satınalma Uzmanı ana şablon rolü (Kullanıcıya atanmaz) [^5]. |
| `Z_S4_MM_D_BUYER_1000` | Derived (Türetilmiş) | MM | 1000 Şirket Kodu için türetilmiş Satınalma Uzmanı rolü [^5]. |
| `Z_S4_MM_D_BUYER_2000` | Derived (Türetilmiş) | MM | 2000 Şirket Kodu için türetilmiş Satınalma Uzmanı rolü [^5]. |
| `Z_S4_FI_S_AP_CLERK` | Single (Tekil) | FI | Satıcı fatura kayıt personeli temel işlem rolü [^1]. |
| `Z_S4_MM_C_SR_BUYER_1000` | Composite (Birleşik) | MM | Kıdemli Satınalmacı tekil ve türetilmiş rol paketi [^4]. |
| `Z_S4_BC_S_BASIS_ADMIN` | Single (Tekil) | Basis | Sistem yöneticisi teknik destek rolü [^1]. |

---

### 1.2. Birleşik Rol Stratejisi (Composite Role Strategy) ve "Kombinatoryal Patlama" (Role Explosion) Riski

Geleneksel ABAP yetkilendirme yaklaşımında **Birleşik Rol Stratejisi (Composite Role Strategy)**; menü yetkilerini fonksiyonel tekil rollerde, veri kısıtlarını ise türetilmiş rollerde toplayarak kullanıcıya tek bir üst paket bağlamayı hedefler [^4].

```
                  ┌────────────────────────────────────────────────────────┐
                  │   BİRLEŞİK ROL (COMPOSITE ROLE): Z_S4_MM_C_BUYER_1000  │
                  └──────────────────────────┬─────────────────────────────┘
                                             │ Kullanıcıya atanan üst paket
                     ┌───────────────────────┴───────────────────────┐
                     ▼                                               ▼
      ┌─────────────────────────────┐                 ┌─────────────────────────────┐
      │     FONKSİYONEL MENÜ ROLÜ   │                 │     ORGANİZASYONEL TÜRETİLMİŞ│
      │   (Single Task/Display)     │                 │   (Derived Organizational)  │
      ├─────────────────────────────┤                 ├─────────────────────────────┤
      │ • Z_S4_BASIS_S_DISPLAY_ALL  │                 │ • Z_S4_MM_D_PO_CREATE_1000  │
      │ • Z_S4_MM_S_COMMON_CATALOG  │                 │ • BUKRS = 1000, EKORG = 1000│
      │ • Salt Okunur / ESS Menüsü  │                 │ • Belge Yaratma & Değiştirme│
      └─────────────────────────────┘                 └─────────────────────────────┘
```

> [!CAUTION]
> **Kurumsal Ölçekte Birleşik Rol "Kombinatoryal Patlama" (Combinatorial Explosion) Tehlikesi:**  
> Büyük ölçekli ve çok şirket kodlu holding yapılarında klasik Birleşik Rol mimarisi ciddi bir bakım krizine yol açar:
> * Eğer şirket kodu ve tesis bazında ayrışan her organizasyonel türetilmiş rol (`_1000`, `_2000`, `_3000`...) için ayrı bir Birleşik Rol (`Z_S4_MM_C_BUYER_1000`, `Z_S4_MM_C_BUYER_2000`...) açılırsa; **50 şirket kodu ve 100 iş ailesi bulunan bir grupta 5.000 adet birleşik rol oluşur**.
> * Bu durum SUIM raporlarında, rol revizyonlarında ve rol yaşam döngüsü yönetiminde yönetilemez bir karmaşa yaratır.
> 
> **Modern IGA Çözümü (İş Rolü / Business Role Modeli):**  
> Modern kimlik yönetişimi (Microsoft Entra ID Governance ve SAP IAG) standartlarında, PFCG seviyesinde binlerce yapay Composite rol açmak yerine;
> 1. SAP seviyesinde sadece atomik **Fonksiyonel Tekil Roller** (`Single Roles`) ve **Organizasyonel Türetilmiş Roller** (`Derived Roles`) tutulur.
> 2. Kullanıcıya bu iki katman doğrudan **Microsoft Entra ID Erişim Paketleri (Access Packages)** veya Dinamik Gruplar üzerinden çoklu (N-to-N) olarak bağlanır [^4][^8].
> 3. Böylece sistemdeki toplam rol sayısı 5.000'den 150 civarına düşürülerek rol patlaması (role explosion) kalıcı olarak engellenir.

---

### 1.3. SAP Standart Rolleri (`SAP_*`) vs Müşteri Özel Rolleri (`Z*`, `Y*`) ve Customizing İlkeleri

SAP, sistem kurulumunda yüzlerce hazır standart rol (`SAP_*`) sunar. Ancak bu roller üretim (Production) ortamında doğrudan kullanıcılara atanmamalıdır [^1][^2]:

* **Standart Rollerin Riskleri:** `SAP_*` rolleri sistem yükseltmelerinde (Upgrade / Support Package) SAP tarafından ezilebilir veya değiştirilebilir. Bu durum mevcut yetkilerin bozulmasına veya yetki genişlemesine (Privilege Creep) neden olur [^1].
* **Kopyalama ve Özelleştirme Standardı (Customizing Roles):** Standart rol `PFCG` kopyalama fonksiyonu ile `Z_` veya `Y_` ad alanına alınır [^2].
* **SU24 / SU25 Yetki Veritabanı Entegrasyonu:** Müşteri rollerinde yetki nesnelerinin kontrol göstergeleri (`Check/Maintain` bayrakları) `SU24` üzerinde kurumsal güvenlik ilkelerine göre uyarlanmalı; upgrade sonrası `SU25` adımları eksiksiz yürütülmelidir [^2].

---

### 1.4. SAP S/4HANA Cloud 3-System Landscape (3SL) İş Rolleri, Spaces ve Pages Standardı (SAP Activate S4H_1665)

SAP S/4HANA Cloud ve modern Private Cloud sistemlerinde 3 Sistemli Ortam (Development $\rightarrow$ Test $\rightarrow$ Production - 3SL) kullanılmaktadır. Bu mimaride roller, Fiori Launchpad'in görsel öğeleri olan **Spaces & Pages** ve **Business Catalogs** ile entegre bir şekilde adlandırılmalıdır [^6][^7].

SAP Activate Hızlandırıcısı **S4H_1665** uyarınca kabul edilen isimlendirme kuralı [^7]:

| Öğe Tipi | Adlandırma Formatı | Örnek | Açıklama |
|:---|:---|:---|:---|
| **İş Rolü (Business Role)** | `BR_<FONKSİYON>_<KAPSAM>` | `BR_PURCHASER_TR` | Kullanıcının kurumsal unvanına bağlanan genel iş rolü [^6]. |
| **İş Alanı (Business Space)** | `SP_<MODÜL>_<İŞLEV>` | `SP_MM_PURCHASING` | Fiori Launchpad üzerinde ana sekmeyi temsil eden alan [^7]. |
| **İş Sayfası (Business Page)** | `PG_<MODÜL>_<İŞLEV>_<DETAY>` | `PG_MM_BUYER_OVERVIEW` | Space altında Fiori kutucuklarını (Tiles) barındıran sayfa [^7]. |
| **İş Kataloğu (Business Catalog)** | `BC_<MODÜL>_<İŞLEV>` | `BC_MM_PURCHASING_OP` | İlgili Fiori uygulamalarının yetki nesnelerini açan katalog [^6]. |

---

### 1.5. Acil Erişim Yönetimi (Firefighter / EAM) Rol İsimlendirme ve Yönetişim Standartları

Kritik üretim ortamlarında acil durumlarda (Break-Glass / EAM) sistem müdahalesi yapabilmek için SAP GRC Access Control Firefighter altyapısı kullanılır [^3]. Acil durum hesap ve rollerinde uluslararası güvenlik standartlarına tam uyum şarttır [^3][^22]:

1. **Firefighter Kimliği (FFID) Formatı:**
   - Standart Format: `FFID_<MODÜL>_<SEVİYE>`
   - Örnekler: `FFID_BASIS_ADMIN`, `FFID_FIN_CRITICAL`, `FFID_MM_SUPPORT`.
2. **Firefighter Rolü Formatı:**
   - Standart Format: `Z_FF_<MODÜL>_<AMAC>`
   - Örnekler: `Z_FF_BASIS_ALL`, `Z_FF_S4_DEBUG_CHANGE`.
3. **Firefighter Yönetişim İlkeleri:**
   - **Gerekçe Kodu (Reason Code):** Oturum açılırken geçerli bir IT servis masası çağrı numarası (Jira / ServiceNow bilet no) girilmesi zorunludur [^3].
   - **Denetçi İncelemesi (Controller Review):** Firefighter oturumu kapandığında, kullanıcının çalıştırdığı tüm işlem kodları ve yaptığı tablo değişiklikleri sistem tarafından loglanarak atanan Denetçiye (Controller) onay/inceleme için iş akışı olarak gönderilir [^3].
   - **IPS İstisnası:** Firefighter rolleri kurumsal dizin (Entra ID) üzerinden otomatik olarak provizyon edilmez; IPS dönüşüm filtreleriyle (`abap.role.filter`) merkezi senkronizasyondan hariç tutulur [^3][^10].

---

### 1.6. SAP BTP Rol Koleksiyonu (Role Collection) ve Yetki Deseni Standartları

SAP BTP Subaccount seviyesinde yetkiler doğrudan tekil rollerle değil, `Role Collection` adı verilen koleksiyonlar altında gruplanarak yönetilir [^9][^15].

```
+-------------------------------------------------------------------------+
|                  SAP BTP ROL KOLEKSİYONU İSİMLENDİRME STANDARDI         |
|                                                                         |
|      <UYGULAMA/SERVİS> _ <MODÜL/FONKSİYON> _ <YETKİ_SEVİYESİ>            |
|      ─────────────────   ─────────────────   ────────────────            |
|       BTP Servis Kodu     Operasyonel İşlev    Admin/User/Viewer         |
+-------------------------------------------------------------------------+
```

* **Standart Örnekler:**
  - `BTP_SAC_BI_ANALYST` : SAP Analytics Cloud analist seviyesi rol koleksiyonu [^9].
  - `BTP_BUILD_PROCESS_APPROVER` : SAP Build Process Automation onaycı koleksiyonu [^15].
  - `BTP_CAP_FIN_VIEWER` : BTP üzerinde koşan CAP uygulaması finans izleme rolü [^15].
  - `BTP_AI_LAUNCHPAD_ADMIN` : SAP AI Launchpad uygulaması üzerinden AI Core modellerini ve deployment'ları yöneten admin koleksiyonu [^12][^15].
  - `BTP_ADMIN_SUBACCOUNT` : Subaccount seviyesi platform operasyon koleksiyonu [^15].

> [!NOTE]
> **SAP AI Core Yetkilendirme Ayrımı:**  
> SAP AI Core çekirdek altyapısı doğrudan BTP Subaccount seviyesinde fabrika teslimi bir Role Collection barındırmaz. AI Core API'lerine doğrudan erişimler Service Key Scopes (`ai-core.admin`, `ai-core.write`, `ai-core.read`) üzerinden yönetilir. Son kullanıcı ve uzmanların görsel arayüz erişimleri ise BTP üzerinde koşan **SAP AI Launchpad** uygulamasının Rol Koleksiyonları (`BTP_AI_LAUNCHPAD_ADMIN`, `AI Launchpad Viewer`) aracılığıyla sağlanır [^12][^15].

---

### 1.7. Microsoft Entra ID Güvenlik Grubu ve Yetkilendirme Paketi Standartları

Merkezi dizin üzerinden SAP rollerini tetiklemek için kullanılan güvenlik grupları doğrudan hedef sistem, ortam ve rol tipi ile eşleşmelidir [^8][^10]:

* **Standart Format:** `SG_SAP_<HEDEF_SİSTEM>_<MODÜL>_<İŞLEV>_<KISIT>`
* **Örnekler:**
  - `SG_SAP_S4P_MM_BUYER_1000` $\rightarrow$ S/4HANA Production, MM Modülü, 1000 Şirket Kodu Satınalma Grubu [^8].
  - `SG_SAP_S4P_FI_AP_CLERK` $\rightarrow$ S/4HANA Production, FI Borç Muhasebesi Rol Grubu [^8].
  - `SG_SAP_BTP_SAC_ANALYST` $\rightarrow$ SAP BTP, Analytics Cloud Analist Grubu [^9].
  - `SG_SAP_ALL_EMPLOYEES` $\rightarrow$ Tüm çalışanlar için geçerli temel ESS (Employee Self-Service) doğum hakkı grubu [^8].
* **Entra ID Governance Yetkilendirme Paketi (Access Package):**
  - Paket Adı: `AP_SAP_S4P_SATINALMA_UZMANI`
  - İçerik: `SG_SAP_S4P_MM_BUYER_1000` grubu + `SG_SAP_ALL_EMPLOYEES` grubu + Fiori Launchpad erişim lisansı [^8].

---

## 2. Rol Gruplarının Varsayılan Atanma Değerleri ve Çok Kiracılı Dizin Mimarisi

Kullanıcıya veya gruba sistemde rol atanırken devreye giren varsayılan parametreler, tarih sınırları ve nitelik kuralları sistem tutarlılığını sağlar [^8][^14].

### 2.1. SAP S/4HANA SU01 ve AGR_USERS Varsayılan Parametreleri

Kullanıcıya bir PFCG rolü bağlandığında arka planda `AGR_USERS` tablosuna kayıt atılır [^1][^2]. Standart atanma değerleri şunlardır:

```
+-------------------------------------------------------------------------+
|                  AGR_USERS TABLOSU VARSAYILAN DEĞERLERİ                 |
|                                                                         |
|  UNAME (Kullanıcı Adı)   : AHMET.YILMAZ                                 |
|  AGR_NAME (Teknik Rol)   : Z_S4_MM_D_BUYER_1000                         |
|  FROM_DAT (Başlangıç)    : SY-DATUM (Örn: 2026-11-03 - Atama Günü)      |
|  TO_DAT (Bitiş Tarihi)   : 31.12.9999 (Sonsuz Geçerlilik)               |
|  EXCLUDE (Hariç Tutma)   : ' ' (Aktif Kullanım)                         |
|  ORG_FLAG (Org Bayrağı)  : ' ' (Doğrudan Atama)                         |
+-------------------------------------------------------------------------+
```

1. **Geçerlilik Başlangıç ve Bitiş Tarihleri (`FROM_DAT` / `TO_DAT`):**
   - Varsayılan başlangıç tarihi günün tarihidir (`SY-DATUM`).
   - Varsayılan bitiş tarihi `31.12.9999` olarak atanır. Bu kural ABAP veri sözlüğü (`AGR_USERS-TO_DAT`) ve `PRGN_CUST` tablosundaki standart yetki parametrelerinden kaynaklanır [^1][^2].
   - SCIM 2.0 üzerinden atamalarda bu değerler JSON yükündeki `validFrom` ve `validTo` niteliklerinden beslenir; boş bırakılırsa hedef sistem varsayılanları devreye girer [^18].
2. **Kullanıcı Tipi (`USTYP` - User Type):**
   - Standart personel için kullanıcı tipi varsayılan olarak `A` (Dialog) atanır [^1][^2].
   - BTP, Cloud Connector ve teknik API entegrasyonları için açılan hesaplar `B` (System User) veya `C` (Communication User) olarak yapılandırılmalıdır [^2][^13].
3. **Kullanıcı Parametreleri (SU3 / `USR05` Parametreleri):**
   - Kullanıcının işlem kodlarında varsayılan olarak göreceği organizasyon parametreleri (`BUK`: Şirket Kodu, `WRK`: Tesis, `EKO`: Satınalma Org).
   - **Teknik İlke:** SCIM 2.0 çekirdek şeması SU3 parametrelerini doğrudan yönetmez [^8][^18]. Bu nedenle veri kısıtları kullanıcı parametresiyle değil, türetilmiş rollerle (Derived Roles) garanti altına alınır [^5].

---

### 2.2. SAP IPS Varsayılan Nitelik Atama Kuralları (Transformations `defaultValue`)

IPS, kaynak sistemden (Entra ID / SuccessFactors) veri çekerken bazı alanların boş gelmesi durumunda sistemin hata vermesini önlemek için `defaultValue` mekanizmasını kullanır [^10][^14]:

```json
[
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
]
```

---

### 2.3. SAP Cloud Identity Services - Identity Directory ve Çok Kiracılı Paket (Bundle) Mimarisi

SAP Cloud Identity Services (IAS/IPS), tüm SAP bulut ekosisteminde merkezi kimlik dizini (Identity Directory) olarak görev yapar [^14].

```
                ┌────────────────────────────────────────────────────────┐
                │          SAP CLOUD IDENTITY SERVICES (IAS/IPS)         │
                │        Central Identity Directory (SCIM 2.0 Hub)       │
                └──────────────────────────┬─────────────────────────────┘
                                           │
         ┌─────────────────────────────────┼─────────────────────────────────┐
         ▼                                 ▼                                 ▼
┌──────────────────┐             ┌──────────────────┐             ┌──────────────────┐
│  S/4HANA Cloud   │             │  Cloud Identity  │             │  SAP Build Work  │
│  Public Edition  │             │  Access Gov.     │             │  Zone Advanced   │
│  Bundle Tenant   │             │  (IAG) Bundle    │             │  Bundle Tenant   │
└──────────────────┘             └──────────────────┘             └──────────────────┘
```

1. **Paket Kiracı Tipleri (Tenant Bundles):**
   - Müşterinin sahip olduğu SAP lisans paketine göre Identity Authentication servisi farklı yeteneklerle gelir: *S/4HANA Cloud Bundle*, *Cloud Identity Access Governance (IAG) Bundle*, *SAP Build Work Zone Bundle* [^14].
2. **Merkezi Kimlik Dizini (Identity Directory):**
   - Kullanıcıların `userUuid` bazında tekilleştirildiği, SCIM 2.0 standartlarında REST servisleri sunan bulut veri deposudur [^14][^18].
3. **Yetkilendirme İlkeleri (Authorization Policies):**
   - IAS konsolu üzerinden kullanıcıların oturum açma anında hangi uygulamalara erişebileceği, kaynak IdP niteliklerine göre dinamik olarak sınırlandırılabilir [^14].

---

### 2.4. SAP BTP Varsayılan Rol Koleksiyonu Ataması ve Shadow User Yaşam Döngüsü

SAP BTP Subaccount üzerinde OIDC / SAML 2.0 Trust yapılandırmasında **Default Role Collections** özelliği bulunur [^10][^15]:

* Kullanıcı kurumsal IdP (Entra ID) üzerinden doğrulandığında, gruplarında özel bir yetki olmasa dahi bu varsayılan koleksiyonları otomatik olarak alır [^10].
* **Önerilen Varsayılan Koleksiyonlar:**
  - `Subaccount User` : Temel platform oturum açma izni [^15].
  - `Launchpad_Viewer` : Fiori Cloud Portal ana sayfasını görüntüleme yetkisi [^15].
* **Gölge Kullanıcı (Shadow User) Davranışı:** BTP, kullanıcının profilini yerel veritabanında saklamaz; doğrulama anında `userUuid` ve `mail` niteliğiyle dinamik bir gölge kayıt oluşturur [^14][^15].

---

### 2.5. Microsoft Entra ID Governance ile Doğum Hakkı (Birthright) ve Yetkilendirme Paketleri

Microsoft Entra ID Governance, SAP uygulamalarına yönelik erişimleri kurumsal politikalara göre otomatikleştiren gelişmiş IGA yetenekleri sunar [^8]:

1. **Doğum Hakkı (Birthright) Atamaları:** SuccessFactors'tan işe giriş kaydı geldiğinde kullanıcının departman ve unvanına göre otomatik tetiklenen dinamik kural grupları [^8].
2. **Erişim Paketleri (Access Packages):** Kullanıcının tek bir onay iş akışıyla SAP S/4HANA, BTP ve ilişkili lisansları topluca talep edebilmesini sağlayan paket yapısı [^8].

---

## 3. Uçtan Uca Çift Yönlü Senkronizasyon, Mock Kullanıcı Denemeleri ve SCIM Doğrulaması

Uçtan uca mimaride çift yönlü (Bidirectional) yetki aktarımı, sistemlerin senkron kalmasını sağlarken döngüsel çakışma (Circular Sync Loop) riskini de beraberinde getirir [^8][^19].

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

### 3.1. Çift Yönlü Senkronizasyon Riskleri, Döngüsel Çakışma (Circular Loop) ve SSoT İlkeleri

1. **Döngüsel Çakışma (Circular Sync Loop) Tehlikesi:**
   - Eğer Sistem A, Sistem B'ye kullanıcı yazarken; Sistem B de Sistem A'daki aynı kayıtları güncellemeye kalkarsa sonsuz bir döngü oluşur ve API kotası tükenir [^19].
   - **Tekil Güven Kaynağı (Single Source of Truth - SSoT) İlkesi:**
     - Kullanıcı kimlik verisinin ve yetki grubu üyeliğinin tek kaynağı **Entra ID / SuccessFactors** olmalıdır [^8].
     - S/4HANA ve BTP sistemleri kullanıcı nitelikleri için yalnızca **Hedef (Target)** olarak konumlandırılmalıdır [^8][^10].
     - Geri yönde aktarım yapılacaksa bu yalnızca **Rol Envanteri Okuma (Role Discovery)** ile sınırlandırılmalı; kullanıcı atamaları geri yönde ezilmemelidir [^8].
2. **SCIM Sürüm Denetimi ve ETag (Concurrency Control):**
   - RFC 7644 Bölüm 3.14 uyarınca eşzamanlı çakışmaları önlemek için HTTP `ETag` başlıkları kullanılır [^19]. Bir kaydın versiyonu uyuşmadığında sistem işlemi `HTTP 412 Precondition Failed` ile reddeder.

---

### 3.2. Gerçek Zamanlı (Real-Time) Provizyon vs Zamanlanmış (Scheduled) ETL Senkronizasyonu

Kurumsal yapılarda kimlik aktarımı iki farklı modda çalışır [^14]:

| Karşılaştırma Kriteri | Zamanlanmış Senkronizasyon (Scheduled Job) | Gerçek Zamanlı Provizyon (Real-Time Provisioning) |
|:---|:---|:---|
| **Çalışma Prensibi** | IPS üzerinde her 20-30 dakikada bir çalışan toplu (batch) delta işi. | Kullanıcı verisi değiştiği anda webhooks / event-driven tetikleme [^14]. |
| **Gecikme (Latency)** | 5 ila 30 dakika arasında bekleme süresi oluşabilir. | 1 ila 5 saniye içinde hedef sistemlere yansır. |
| **Sistem Yükü** | Düzenli aralıklarla API sorguları yapar. | Yalnızca olay gerçekleştiğinde hedef sistem API'sini çağırır. |
| **Kullanım Alanı** | Standart JML işe giriş ve görev değişimleri. | Acil hesap dondurma, güvenlik ihlali ve anlık parola sıfırlama [^14]. |

---

### 3.3. Yerel Yetki Koruma ve Rol İstisna Filtreleri (`abap.role.filter` & `ips.delete.threshold`)

Merkezi senkronizasyonda yerel yetkilerin korunması için IPS üzerinde iki kritik sistem özelliği (System Properties) yapılandırılmalıdır [^10][^14]:

1. **`abap.role.filter`:**
   SAP Cloud Identity Services (IPS) hedef sistem konfigürasyonunda yer alır [^10]. S/4HANA sisteminde Basis tarafından doğrudan atanmış acil durum (`Z_FF_*`) veya sistem rollerinin IPS senkronizasyonunda hedef sistemden geri alınmasını (revoke) engeller.
2. **`ips.delete.threshold`:**
   Kurumsal dizinde yanlışlıkla bir grubun silinmesi halinde SAP tarafında toplu yetki silinmesini engelleyen güvenlik sigortasıdır (Varsayılan eşik: 10 kayıt) [^10].

---

### 3.4. Mock Kullanıcı ve Test Senaryoları Tasarımı

Entegrasyon testlerinde canlı veriyi riske atmamak için 3 farklı mock kullanıcı profili kullanılır [^8][^10]:

| Mock Kullanıcı | Senaryo Tipi | Test Edilecek Nitelikler | Beklenen Sistem Davranışı |
|:---|:---|:---|:---|
| `TEST_MOCK_USER_01` | Standart Giriş (Day-1 Birthright) | `SG_SAP_S4P_MM_BUYER_1000`<br>`SG_SAP_ALL_EMPLOYEES` | S/4HANA'da hesap açılır; `Z_S4_MM_D_BUYER_1000` ve `Z_S4_BASIS_S_EMPLOYEE_ESS` rolleri bağlanır [^8]. |
| `TEST_MOCK_USER_02` | Görev Değişimi (Mover Testi) | MM grubundan çıkarılıp `SG_SAP_S4P_FI_AP_CLERK` grubuna eklenir. | MM rolü geri alınır; FI rolü eklenir. Mevcut SU01 temel parametreleri korunur [^8]. |
| `TEST_MOCK_USER_03` | Acil Durum / Filtre Testi | Kullanıcıya S/4HANA'da doğrudan `Z_FF_BASIS` rolü atanır; AD grubunda bu rol yoktur. | IPS çalıştığında `Z_FF_*` filtresi sayesinde acil durum rolü silinmez, korunur [^3][^10]. |

---

### 3.5. Genişletilmiş SCIM 2.0 Mock JSON İstek Yükleri (POST, PATCH, Delta)

IETF SCIM 2.0 standartlarına (RFC 7643 / RFC 7644) ve SAP Identity Directory şemasına tam uyumlu test yükleri [^14][^18][^19]:

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
    "validFrom": "2026-11-03T00:00:00Z",
    "validTo": "9999-12-31T23:59:59Z"
  }
}
```

#### 2. Mock Grup ve Rol Atama / Geri Alma İsteği (Delta PATCH `/Groups`)

```json
{
  "schemas": [
    "urn:ietf:params:scim:api:messages:2.0:PatchOp"
  ],
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
    },
    {
      "op": "remove",
      "path": "members[value eq \"b2c3d4e5-f6a7-8b9c-0d1e-2f3a4b5c6d7e\"]"
    }
  ]
}
```

---

### 3.6. Test Metodolojisi, IPS Simülasyon Doğrulaması ve Yetki Kayması (Drift) Analizi

1. **İzole Test Ortamı:** Mock denemeleri doğrudan Canlı (Production) üzerinde değil; Quality (QA/Test) subaccount ve S/4HANA Test istemcisinde yürütülmelidir.
2. **IPS Simulation Mode Doğrulaması:** IPS yönetim konsolundan **Run Simulation** tetiklenerek hedef sistemde oluşacak `CREATE`, `UPDATE`, `DELETE` işlemleri canlıya yazılmadan raporlanır [^10].
3. **Yetki Kayması (Continuous Drift) Tespiti:** S/4HANA üzerinde Basis tarafından yerel açılan yetkiler ile Entra ID grupları arasındaki sapmalar haftalık mutabakat raporlarıyla denetlenmelidir [^8][^11].

---

## 4. SAP BTP Bağlantı Mimarisi, Destination Service ve Principal Propagation Mekanizmaları

BTP üzerinde koşan bulut uygulamalarının şirket içi (On-Premise) S/4HANA sistemleri veya diğer bulut servisleriyle haberleşirken oturum açmış kullanıcının kimliğini ve yetki bağlamını kaybetmeden iletmesi, Sıfır Güven (Zero Trust) mimarisinin temel taşıdır [^13][^23].

```
┌─────────────────┐       SAML / JWT Bearer      ┌─────────────────┐  X.509 Ephemeral Cert ┌─────────────────┐
│ SAP BTP App     │ ───────────────────────────► │ SAP Cloud       │ ────────────────────► │ SAP S/4HANA     │
│ (Cloud Foundry) │       via Destination        │ Connector (SCC) │      RFC / HTTPS      │ (STRUST/CERTRULE│
└─────────────────┘                              └─────────────────┘                       └─────────────────┘
```

### 4.1. Buluttan Şirket İçi Sistemlere (Cloud-to-On-Premise) Kimlik Aktarımı, STRUST ve ICM Güven Zinciri

BTP bulut ortamından yerel ağdaki SAP S/4HANA sistemine erişimde **SAP Cloud Connector (SCC)** güvenli bir ters tünel (reverse tunnel) açar [^13]. Ancak bu kimlik aktarımının başarılı olabilmesi için S/4HANA altyapısında iki kritik önkoşul katmanı tamamlanmış olmalıdır:

1. **Kısa Ömürlü X.509 İstemci Sertifikaları (Short-Lived Ephemeral Certificates):**
   - Kullanıcı BTP arayüzünde işlem yaptığında, Cloud Connector kullanıcının kimlik bilgisine (`mail`, `userName` veya `userUuid`) karşılık gelen ve yalnızca birkaç dakika geçerli olan geçici bir X.509 sertifikası üretir [^13].
2. **S/4HANA STRUST Sertifika Güven Zinciri (Önkoşul Katmanı):**
   - Cloud Connector'ın ürettiği sertifikayı imzalayan CA kök/ara sertifikası S/4HANA üzerinde `STRUST` işlem kodunda `SSL Server Standard` veya `cAVK` (Client Certificate CA) havuzuna eklenmelidir [^13].
   - Sertifika güven zinciri doğrulanmadığı takdirde HTTPS el sıkışması anında başarısız olur.
3. **S/4HANA ICM Profil Parametreleri Yapılandırması:**
   - Web Dispatcher veya SAP Internet Communication Manager (ICM) katmanında istemci sertifikalarının güvenli kabul edilmesi için profil parametreleri eksiksiz girilmelidir [^13]:
     * `icm/HTTPS/trust_client_certs = TRUE`
     * `icm/accept_forwarded_cert_via_http = TRUE` (Eğer Cloud Connector ile S/4HANA arasında bir ters proxy / yük dengeleyici varsa).
   - Bu parametreler yapılandırılmadığında ICM katmanı gelen sertifikayı güvenilmez sayarak doğrudan `HTTP 401 Unauthorized` veya `HTTP 403 Forbidden` ile bağlantıyı reddeder.
4. **S/4HANA Kural Tabanlı Sertifika Eşleme (`CERTRULE`):**
   - Sertifika güven zincirini geçen istekte, `CERTRULE` işlem kodu üzerinden tanımlanan kural ile sertifikadaki e-posta (`CN=${email}`) kullanıcının `SU01` kullanıcı adına dinamik eşleştirilir [^13].
5. **Dış Kimlik Tablosu (`USREXTID`):**
   - Alternatif olarak `EXTIDDN` türünde dış kimlik tablosuna kayıt atılarak kullanıcı bazlı birebir eşleme sağlanabilir.
   - Bu sayede kullanıcının BTP üzerinden başlattığı bir satınalma siparişi S/4HANA'ya ulaştığında, işlemi yapan kişi olarak BTP teknik kullanıcısı değil, **Ahmet Yılmaz'ın kendi kullanıcı adı** loglanır [^13][^22].

---

### 4.2. Buluttan Buluta (Cloud-to-Cloud) Kimlik Aktarımı ve Belirteç (Token) Akışları

Bulut servisleri arasındaki haberleşmede IETF standartlarında modern belirteç (token) değişim protokolleri kullanılır [^13][^20][^21]:

1. **SAML 2.0 Bearer Assertion Flow (RFC 7522):**
   - Cloud Foundry üzerindeki bir uygulamanın SAP S/4HANA Cloud veya SuccessFactors sistemine kullanıcının kimliğiyle REST/OData çağrısı yapmasını sağlar [^13][^20].
2. **OAuth 2.0 JWT Bearer Flow (RFC 7523):**
   - SAP Cloud Identity Services ekosisteminde iki mikroservis arasında JSON Web Token (JWT) aktarımı için kullanılır [^13][^21].
3. **OAuth 2.0 Token Exchange (RFC 8693):**
   - Kullanıcının mevcut erişim belirtecinin (`Subject Token`), hedef servisin kabul edeceği formatta yeni bir belirteçle takas edilmesini sağlar (`urn:ietf:params:oauth:grant-type:token-exchange`) [^13].
4. **Teknik Kullanıcı vs İş Kullanıcısı Ayrımı:**
   - Arka plan senkronizasyonlarında `Client Credentials` akışı ile teknik servis kullanıcısı kullanılırken; kullanıcı etkileşimli ekranlarda mutlaka `Principal Propagation` zorunlu tutulmalıdır [^13].

---

### 4.3. Destination Service Güvenliği, Kimlik Doğrulama Tipleri ve Application Router (`approuter`) Mimarisi

SAP BTP **Destination Service**, uygulamaların hedef sistem bağlantı URL'lerini, kimlik doğrulama modellerini ve sertifikalarını merkezi olarak yönetir [^13]:

* **Kimlik Doğrulama Tipleri:**
  - `PrincipalPropagation` : Cloud Connector üzerinden şirket içine güvenli kimlik aktarımı [^13].
  - `OAuth2SAMLBearerAssertion` : Bulut sistemlerine SAML bilet takası [^13][^20].
  - `OAuth2UserTokenExchange` : Mikroservisler arası kullanıcı bağlamı takası [^13].
* **Şifreleme ve Kasa Güvenliği:**
  - Destination servisindeki tüm parolalar ve özel anahtarlar AES-256 ile şifrelenmiş donanımsal güvenlik modüllerinde (HSM) korunur [^13].
* **Application Router (`approuter`) İşlevi:**
  - BTP mimarisinde gelen istekleri karşılayan, kullanıcının oturum JWT belirtecini alan, CSRF token koruması sağlayan ve yetkili isteği hedef mikroservise yönlendiren çekirdek bileşendir [^13][^15].

---

## 5. SAP BTP Çekirdek IAM Mimarisi ve Platform / İş Kullanıcısı Yönetişimi

SAP BTP ortamında kullanıcı yönetimi ve yetkilendirme, klasik ABAP sistemlerinden farklı olarak çok katmanlı bulut ilkelerine dayanır [^15].

```
+-------------------------------------------------------------------------+
|                  SAP BTP YETKİLENDİRME HİYERARŞİSİ                      |
|                                                                         |
|   ROLE COLLECTION (Kullanıcıya / IdP Grubuna Atanan Paket)              |
|        └── ROLES (Uygulama Rolleri)                                     |
|             └── ROLE TEMPLATES (Geliştirici Tarafından Tanımlanan Şablon|
|                  └── SCOPES (Yetki Kapsamları: Read, Write, Admin)      |
+-------------------------------------------------------------------------+
```

### 5.1. Platform Kullanıcıları (Platform Users) vs İş Kullanıcıları (Business Users) Ayrımı

SAP BTP'de iki bağımsız kullanıcı katmanı bulunur [^15]:

1. **Platform Kullanıcıları (Platform Users):**
   - BTP Cockpit üzerinde Global Account, Directory veya Subaccount seviyesinde altyapı yönetimi yapan, servis oluşturan veya kod geliştiren teknik uzmanlardır (Örn: Subaccount Administrator, Cloud Foundry Space Developer) [^15].
2. **İş Kullanıcıları (Business Users):**
   - BTP üzerinde barındırılan kurumsal iş uygulamalarını (SAP Analytics Cloud, Build Apps, özel CAP uygulamaları) kullanan son kullanıcılardır [^15].

---

### 5.2. BTP Yetkilendirme Hiyerarşisi (`xs-security.json` & XSUAA)

BTP üzerinde çalışan bulut uygulamalarında yetkiler **XSUAA (Extended Services User Account and Authentication)** servisi ve `xs-security.json` dosyası ile tanımlanır [^15]:

```json
{
  "xsappname": "sap-iam-governance-app",
  "scopes": [
    {
      "name": "$XSAPPNAME.Viewer",
      "description": "Finansal raporları izleme yetkisi"
    },
    {
      "name": "$XSAPPNAME.Approver",
      "description": "Satınalma siparişlerini onaylama yetkisi"
    }
  ],
  "role-templates": [
    {
      "name": "FinanceViewerTemplate",
      "description": "Finans izleyici şablonu",
      "scope-references": [
        "$XSAPPNAME.Viewer"
      ]
    }
  ]
}
```

* **Deklaratif Yetkilendirme (CAP CDS `@restrict`):**
  SAP Cloud Application Programming Model (CAP) katmanında veritabanı seviyesinde nitelik tabanlı erişim denetimi (ABAC) uygulanır:
  `@restrict: [{ grant: 'READ', to: 'Viewer', where: 'companyCode = $user.companyCode' }]` [^15].

---

### 5.3. Kurumsal Kimlik Sağlayıcı (Corporate IdP) Federasyonu ve Dinamik Rol Eşleme

BTP Subaccount seviyesinde kurumsal IdP (Microsoft Entra ID) ile doğrudan veya SAP IAS üzerinden federasyon kurulur [^10][^14][^15]:

* **Koleksiyon Eşleme (Role Collection Mapping):**
  Entra ID üzerindeki `SG_SAP_BTP_SAC_ANALYST` grubu, BTP Cockpit üzerinde `BTP_SAC_BI_ANALYST` rol koleksiyonuna eşlenir [^9][^10].
* **Dinamik Atama:** Kullanıcı oturum açtığında SAML Assertion veya OIDC ID Token içerisindeki `Groups` claim'i okunarak kullanıcının BTP yetkileri anında verilir; sistem yöneticisinin BTP üzerinde kullanıcı açmasına gerek kalmaz (JIT Federation) [^10][^15].

---

### 5.4. Çoklu Ortam (Multi-Environment) Yetki Mimarisi

SAP BTP, farklı çalıştırma ortamlarında bağımsız yetkilendirme modelleri uygular [^15]:

1. **Cloud Foundry Ortamı:** Org ve Space hiyerarşisi (`Org Manager`, `Space Developer`, `Space Auditor`) [^15].
2. **Kyma Ortamı:** Kubernetes tabanlı container ortamı; Kubernetes RBAC (`Role`, `ClusterRole`, `RoleBinding`) ile OIDC token doğrulaması entegre çalışır [^15].
3. **ABAP Ortamı (Steampunk):** Bulut üzerinde ABAP kodu koşturan ortam; iş katalogları ve iletişim kullanıcıları (`Communication Users`) ile korunur [^15].

---

## 6. Yeni Nesil Kurumsal Yapay Zeka (AI), CDS DCL ve SAP Joule IAM Yönetişimi

Kurumsal üretken yapay zeka ve makine öğrenimi modellerinin devreye alınmasında yetki sınırlarının korunması ve veri sızıntısının önlenmesi en kritik güvenlik önceliğidir [^12][^14][^16].

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SAP AI CORE MULTI-TENANCY                        │
│                                                                         │
│   RESOURCE GROUP: rg-finance-prod      RESOURCE GROUP: rg-hr-confidential│
│   ├── Dataset: General Ledger Data     ├── Dataset: Payroll Records     │
│   ├── Deployment: GPT-4o Finetuned     ├── Deployment: Llama-3 HR Agent │
│   └── Scopes: ai-core.read (Finance)   └── Scopes: ai-core.read (HR)    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.1. SAP AI Core Çok Kiracılılık (Multi-Tenancy) ve Kaynak Grubu (Resource Group) İzolasyonu

SAP AI Core üzerinde yürütülen makine öğrenimi işleri ve LLM dağıtımları **Resource Groups** mekanizması ile mantıksal olarak birbirinden izole edilir [^12]:

* **Kaynak Gruplarının Rolü:** Her kurumsal birim veya müşteri kiracısı için ayrı bir kaynak grubu oluşturulur (Örn: `rg-finance-prod`, `rg-hr-confidential`) [^12].
* **İzolasyon Kapsamı:** Bir kaynak grubunda tanımlanan veri setleri (Datasets), model dağıtımları (Deployments) ve çalıştırma komutları (Executions) diğer kaynak grupları tarafından kesinlikle görülemez ve çağrılamaz [^12].

---

### 6.2. AI Core Servis Anahtarları (Service Keys) ve OAuth 2.0 RBAC Yetki Kapsamları

SAP AI Core API servislerine doğrudan erişim BTP Service Key üzerinden üretilen OAuth 2.0 belirteçleriyle sağlanır [^12]:

* **Çekirdek RBAC Kapsamları (Scopes):**
  - `ai-core.admin` : Kaynak grupları oluşturma, kota belirleme ve altyapı yönetimi [^12].
  - `ai-core.write` : Yeni model dağıtımı yapma ve eğitim işleri (pipeline) başlatma [^12].
  - `ai-core.read` : Model çıktılarını sorgulama ve inferans alma [^12].
* **Yetki Başlığı (Header Control):**
  API çağrısı yapılırken hedef kaynak grubunun belirtilmesi zorunludur:
  `AI-Resource-Group: rg-finance-prod` [^12].

---

### 6.3. Paylaşımlı Kaynak Grupları vs İzole Kiracı Grupları Yönetişimi

* **Shared Resource Group (Paylaşımlı Grup):** Ortak temel LLM modellerinin (Generative AI Hub) tüm departmanlar tarafından genel sorgular için tüketildiği alan [^12].
* **Dedicated Resource Group (Özel Kiracı Grubu):** Gizli kurumsal verilerle (Bordro, Ar-Ge kodları) eğitilen veya bağlam eklenen (RAG) modellerin yalnızca yetkili kullanıcılar tarafından çağrılabildiği yüksek güvenlikli alan [^12].

---

### 6.4. SAP Joule Yapay Zeka Asistanı, OData RAP ve S/4HANA CDS DCL (Data Control Language) Mimarisi

SAP'nin kurumsal üretken yapay zeka asistanı **Joule**, kullanıcı adına sistemler arası sorgulama ve operasyon yürütür [^14]:

1. **Kullanıcı Kimliği Doğrulaması:** Joule, oturum açan personeli SAP Cloud Identity Services (IAS) üzerinden doğrular [^14].
2. **Kullanıcı Yetki Bağlamının Korunması (Context Propagation):**
   - Joule bir kullanıcı adına S/4HANA veya SuccessFactors verisi çekerken kendi teknik süper yetkisiyle değil, **oturum açmış kullanıcının kimlik belirteciyle (Principal Propagation)** çağrı yapar [^13][^14].
3. **Modern S/4HANA Gerçeği: Core Data Services (CDS) DCL Katmanı:**
   - Joule ve üretken yapay zeka aracıları S/4HANA verilerini klasik transaction'lar üzerinden değil; **Core Data Services (CDS) Views** ve **ABAP RESTful Application Programming Model (RAP)** OData servisleri üzerinden çeker [^16].
   - Burada satır ve alan bazlı veri güvenliği klasik `AUTHORITY-CHECK` komutundan ziyade **CDS DCL (Data Control Language - Access Controls)** katmanında uygulanır [^16]:

```sql
@EndUserText.label: 'Satınalma Siparişleri CDS Yetki Kontrolü'
@MappingRole: true
define role Z_DCL_PURCHASE_ORDER {
  grant select on ZI_PurchaseOrderHeader
    where ( CompanyCode ) = aspect pfcg_auth( M_BEST_BUK, BUKRS, ACTVT = '03' );
}
```

* **DCL Yönetişimi:** Kullanıcının PFCG rolündeki `M_BEST_BUK` yetki nesnesinde şirket kodu 1000 tanımlıysa; Joule arka planda CDS görünümünü sorguladığında veritabanı motoru (SAP HANA) SQL seviyesinde bu satır kısıtlamasını işletir [^16].
* Eğer personelin S/4HANA'da ilgili şirket kodunu veya maaş tablosunu görme yetkisi yoksa, Joule arka uçtan boş veri alır ve personele yetkisiz veri sızdıramaz [^14][^16][^22].

---

### 6.5. Generative AI Hub Ölçümleme (Metering), Tüketim Takibi ve Güvenlik Denetimi

SAP AI Core Generative AI Hub üzerinden tüketilen token ve inferans miktarları kiracı bazında ölçümlenir [^12]:

* **Kota Kontrolü:** Departmanların bütçe aşımını önlemek amacıyla kaynak grubu bazında token tüketim limitleri tanımlanır [^12].
* **Denetim İzi (Audit Logging):** Hangi kullanıcının hangi tarih ve saatte ne kadarlık token tükettiği ve hangi model uç noktasını çağırdığı BTP Audit Log servisine kaydedilir [^12][^22].

---

## 7. Rol, Rol Grubu ve Unvan Değişimlerinde Kayıt Defteri, Revizyon ve SIEM İzleme

Kurumsal denetim standartları (SOX 404, ISO/IEC 27001, KVKK) uyarınca bir yetkinin ne zaman, kim tarafından ve hangi gerekçeyle verildiği geriye dönük tam izlenebilir olmalıdır [^22][^23][^24].

```
+-------------------------------------------------------------------------+
|                  SAP DEĞİŞİKLİK BELGELERİ MİMARİSİ                      |
|                                                                         |
|   SU01 / PFCG / SCIM API  ──►  CDHDR (Değişiklik Başlık Tablosu)        |
|                                 ├── Object Class: USER / IDENTITY       |
|                                 ├── Object ID   : AHMET.YILMAZ          |
|                                 ├── Change No   : 0001248590            |
|                                 ├── User Name   : IPS_PROV_SERVICE      |
|                                 └── Timestamp   : 2026-11-03 14:32:10   |
|                                                                         |
|                                CDPOS (Değişiklik Pozisyon Tablosu)      |
|                                 ├── Table Name  : AGR_USERS             |
|                                 ├── Field Name  : AGR_NAME              |
|                                 ├── Change Type : I (Insert) / D (Delete|
|                                 ├── Old Value   : Z_S4_MM_D_BUYER_1000  |
|                                 └── New Value   : Z_S4_FI_S_AP_CLERK    |
+-------------------------------------------------------------------------+
```

### 7.1. SAP S/4HANA Tarafında Yetki Değişim Kayıt Defteri Tabloları

1. **`CDHDR` (Change Document Header):** Değişikliği yapan kullanıcıyı (`USERNAME`), tarihi (`UDATE`), saati (`UTIME`) ve belge numarasını (`CHANGENR`) tutar [^1][^2][^11].
2. **`CDPOS` (Change Document Position):** Hangi tablonun (`TABNAME`: `AGR_USERS`), hangi alanının (`FNAME`: `AGR_NAME`) değiştiğini; eski ve yeni değer kırılımıyla kaydeder (`CHNGIND`: `I`=Insert, `D`=Delete, `U`=Update) [^1][^2][^11].
3. **`USH02` & `USH04`:** Kullanıcı oturum açma, şifre ve yetki profili değişikliklerinin tarihsel kopyaları [^1][^2][^11].
4. **`AGR_USERS`:** Kullanıcının aktif ve geçmiş tüm rol bağlantılarını `FROM_DAT` ve `TO_DAT` tarihleriyle saklar [^1][^2][^11].

---

### 7.2. SUIM (User Information System) Standart Denetim Raporları

SAP NetWeaver / S/4HANA çekirdek güvenlik (BC-SEC) katmanında yer alan standart SUIM raporları, rol ve kullanıcı hareketlerinin geriye dönük denetimini sağlar [^1][^2]:

* **`RSUSR100N` (Change Documents for Users - SUIM):** Kullanıcı bazında rol ekleme, silme ve hesap kilitleme hareketlerinin raporlandığı standart NetWeaver BC-SEC denetim programıdır [^1].
* **`RSSCD100_PFCG` (Change Documents for Roles - SUIM):** PFCG rolünün kendi içindeki yetki nesnesi, yetki alanı veya menü değişikliklerini geriye dönük izleyen standart BC-SEC denetim raporudur [^1][^2].

---

### 7.3. SAP IPS Senkronizasyon Günlükleri ve Detailed Execution Log

IPS motorunun gerçekleştirdiği her işlem yönetim arayüzünde kayıt altına alınır [^10][^14]:

* **Execution Details Raporu:** `Read Entities`, `Created Entities`, `Updated Entities`, `Deleted Entities`, `Failed Entities` özet sayaçları [^10].
* **Detailed Execution Log (CSV / JSON):** Başarısız olan veya istisnaya takılan işlemlerin HTTP hata kodları (`HTTP 409 Conflict`, `HTTP 400 Bad Request`, `HTTP 404 Not Found`) [^10].

---

### 7.4. Microsoft Entra ID Denetim Günlükleri (Audit Logs)

Rol değişiminin ilk tetiklendiği kurumsal dizin kayıtları [^8][^10]:

* **Kategori:** `GroupManagement` $\rightarrow$ `Add member to group` / `Remove member from group`.
* **Aktör Bilgisi:** SuccessFactors entegrasyon servis hesabı veya İK yetkilisi [^8].

---

### 7.5. SAP for Microsoft Sentinel Sürekli Tehdit İzleme ve Anomali Tespiti

Microsoft Sentinel'in SAP Çözümü, SAP altyapısındaki yetki anomalilerini yapay zeka destekli kurallarla gerçek zamanlı izler [^11]:

1. **Yetkisiz Rol Ataması Uyarısı:** IPS dışındaki bir kullanıcı (örn. yerel Basis hesabı) doğrudan `SU01` veya `PFCG` üzerinden kritik bir rol (`SAP_ALL` veya `Z_FF_*`) atarsa Sentinel anında `Privilege Escalation` güvenlik olayı tetikler [^11].
2. **Firefighter Oturum Anomalileri:** Mesai saatleri dışında veya biletsiz açılan Firefighter oturumları Sentinel panolarında alarm üretir [^3][^11].
3. **Security Audit Log (SAL) Entegrasyonu:** Transaction `SM20` denetim logları Sentinel Log Analytics çalışma alanına aktarılarak şüpheli işlem kodları anlık korele edilir [^11].

---

### 7.6. Noktasal Zamanlı Denetim (Point-in-Time Audit) ve Revizyon Modeli (SOX 404, ISO 27001)

Denetçiler şu soruyu sorar: *"Ahmet Bey 15 Mart 2026 tarihinde hangi SAP yetkilerine sahipti ve bu yetki hangi unvan gerekçesiyle verilmişti?"* [^22][^24]

Bu sorunun cevabını üretmek için 3 katmanlı korelasyon işletilir:
1. **İK Pozisyon Kaydı:** SF / İK sistemindeki pozisyon geçmişi tablosu (15 Mart'taki unvanı).
2. **Dizin Grup Üyeliği:** Entra ID Audit Log üzerinden 15 Mart'taki grup üyelikleri [^8].
3. **S/4HANA Yetki Kaydı:** `AGR_USERS` tablosunda `FROM_DAT <= '20260315' AND TO_DAT >= '20260315'` koşulunu sağlayan roller [^1][^2][^11].

---

## 8. Rol Grubu veya Unvanın Pasifleştirilmesi ve Emekli Edilmesi (Lifecycle Management)

Kurumsal organizasyonlarda süreçler geliştikçe bazı roller veya unvanlar işlevini yitirir. Bu durumlarda rolün veya unvanın kontrollü şekilde emekli edilmesi gerekir [^7][^8].

```
+-------------------------------------------------------------------------+
|                  ROL VE UNVAN EMEKLİLİK YAŞAM DÖNGÜSÜ                   |
|                                                                         |
|   1. DUYURU & İŞARETLEME  ──► [DEPRECATED] Etiketi ve Yeni Atamaya Kapat|
|                                                                         |
|   2. GEÇERLİLİK BİTİRME   ──► AGR_USERS TO_DAT = SY-DATUM (Toplu Çıkış) |
|                                                                         |
|   3. TAMPON BELLEK (PFUD) ──► PFCG_TIME_DEPENDENCY (User Buffer Boşalt) |
|                                                                         |
|   4. AD GRUP TASFİYESİ    ──► SG_SAP_RETIRED_* Yeniden Adlandırma       |
|                                                                         |
|   5. KARANTİNA DÖNEMİ     ──► 90 Gün Pasif Bekletme (İç Kontrol Süresi) |
|                                                                         |
|   6. KALICI SİLME         ──► Rolün PFCG ve AD'den Tamamen Kaldırılması |
+-------------------------------------------------------------------------+
```

### 8.1. Kavramsal Ayrım: Pasifleştirme vs. Emekli Etme

* **Pasifleştirme (Deactivation - Geçici Dondurma):**
  - Rolün veya unvanın yeni personele atanması durdurulur; mevcut kullanıcıların yetkisi korunur veya askıya alınır [^7].
* **Emekli Etme (Deprecation / Retirement - Kalıcı Tasfiye):**
  - Rolün kurumsal geçerliliği kalıcı olarak sonlandırılır. Kullanıcılar alternatif yeni rollere taşınır ve karantina süresi sonunda teknik rol sistemden silinir [^7][^8].

---

### 8.2. SAP S/4HANA Tarafında Kademeli Rol Emeklilik Prosedürü ve User Buffer (`PFUD`) Yönetimi

Rol emeklilik süreci, kurumsal iç kontrol en iyi uygulamaları (Best Practices) ve SAP yetki tampon belleği standartları uyarınca 5 aşamada işletilmelidir [^1][^17]:

1. **Aşama 1: Rol Açıklamasının Güncellenmesi ve Kilitlenmesi:**
   - PFCG üzerinde rol tanımına `[DEPRECATED - KULLANILMAZ]` ibaresi eklenir ve menü kilitlenir.
2. **Aşama 2: Toplu Geçerlilik Bitiş Tarihi Güncellemesi (`Mass Role Expiration`):**
   - Aktif personellerin `AGR_USERS` tablosundaki bitiş tarihi bugünün tarihine çekilir (`TO_DAT = SY-DATUM`) [^1].
3. **Aşama 3: Yetki Tampon Belleği (User Buffer) Senkronizasyonu (`PFCG_TIME_DEPENDENCY`):**

> [!CRITICAL]
> **UST04 / User Buffer Senkronizasyon Gecikmesi ve Güvenlik Açığı:**  
> Bir rolün `AGR_USERS` tablosundaki bitiş tarihinin bugüne çekilmesi (`TO_DAT = SY-DATUM`), yetkinin hemen o saniye kullanıcının elinden alındığı anlamına gelmez!  
> SAP sisteminde kullanıcının anlık çalıştırdığı yetkiler veritabanı tablosundan değil, **Yetki Tampon Belleği (User Buffer - Tablo `UST04` / `USR04`)** üzerinden okunur.  
> * **Risk:** Sistemde `PFCG_TIME_DEPENDENCY` arka plan toplu işi (Rapor: `RHAUTUSR_COMPARE` / İşlem Kodu: `PFUD`) çalışana kadar kullanıcının tampon belleği boşaltılmaz. Kullanıcı açık oturumunda geçerliliği biten rolün yetkilerini kullanmaya devam eder.  
> * **Zorunlu Operasyon Adımı:** Toplu rol geçerlilik güncellemesi yapıldığı anda Basis ekibi tarafından `PFUD` (Rapor: `RHAUTUSR_COMPARE`) hemen manuel olarak tetiklenmeli veya sistemde bu toplu işin her 2-4 saatte bir düzenli çalıştığı doğrulanmalıdır [^17].

4. **Aşama 4: 90 Günlük Kurumsal Karantina Süresi (Quarantine Period):**
   - Bu süre bir SAP Activate kuralı değil, kurumsal iç kontrol ve denetim standardıdır.
   - Rol asgari 90 gün boyunca sistemde pasif ve yetkisiz halde bekletilir; beklenmeyen bir operasyonel aksama anında tarih uzatılarak hızlı geri dönüş sağlanabilir.
5. **Aşama 5: Kalıcı Tasfiye ve Arşivleme:**
   - Karantina süresi dolduğunda rol yetki dökümü PDF/metin olarak arşivlenir ve rol PFCG üzerinden silinir [^1].

---

### 8.3. Microsoft Entra ID ve IPS Tarafında Tasfiye Prosedürü

1. **Grup İsimlendirmesinin Değiştirilmesi:**
   - Emekli edilen güvenlik grubu silinmeden önce `SG_SAP_RETIRED_MM_BUYER_1000` olarak adlandırılır [^8].
2. **IPS Kaynak Filtresi:**
   - IPS dönüşüm filtresine retired öneki eklenerek senkronizasyon dışına alınır:
     `displayName startswith 'SG_SAP_' and not (displayName startswith 'SG_SAP_RETIRED_')` [^10].

---

### 8.4. Microsoft Entra ID Governance Yaşam Döngüsü İş Akışları (Lifecycle Workflows) ve Yetki Gözden Geçirme

* **Lifecycle Workflows:** İşten ayrılan (Leaver) bir çalışanın SAP gruplarının anında dondurulmasını ve 30 gün sonra otomatik silinmesini sağlayan zamanlanmış kural akışları [^8].
* **Access Reviews (Erişim Gözden Geçirme):** Kritik SAP rollerine sahip kullanıcıların yöneticilerine 90 günde bir otomatik onay görevi düşürülerek gereksiz yetkilerin geri alınması sağlanır [^8][^22].

---

### 8.5. Yetim Yetki (Orphan Entitlement) Temizliği ve Unvan Geçiş Matrisi

Gruplar filtrelendiğinde S/4HANA üzerinde yetim kalan yetkilerin temizlenmesi için iki yöntem uygulanır [^8][^10]:
* **Planlı Göç:** Personel yeni role taşınırken eski rolün bitiş tarihi `AGR_USERS` üzerinden kapatılır.
* **Toplu Temizlik:** Basis ekibi `SUIM` üzerinden emekli role sahip kullanıcıları listeleyerek toplu silme işlemi yürütür [^1].

---

## 9. Bütünleşik Karar, Mimari Standart ve Araştırma Matrisi

| Konu / Alan | Temel Teknik Bulgu / İlke | Alınması Gereken Önlem / Kurumsal Standart |
|:---|:---|:---|
| **PFCG Rol İsimlendirmesi** | ABAP veritabanında teknik rol ismi azami 30 karakterdir (`AGR_NAME`); `:` karakteri BAPI/RFC regex hatası üretir [^1][^5]. | `Z_<KAPSAM>_<MODÜL>_<TİP>_<İŞLEV>_<ORGANİZASYON>` veya `/NAMESPACE/` standardı bağlayıcı olmalıdır [^1][^2]. |
| **Rol Tasarım Mimarisi** | Şirket kodu bazlı Birleşik Rol tasarımı Kombinatoryal Patlamaya (Role Explosion) yol açar [^4]. | PFCG'de atomik roller tutulmalı; birleşimler **Microsoft Entra ID Erişim Paketleri (Access Packages)** ile sağlanmalıdır [^4][^8]. |
| **Standart vs Müşteri Rolleri** | `SAP_*` rolleri yükseltmelerde ezilebilir ve güvenlik açığı yaratabilir [^1]. | Standart roller kopyalanarak `Z_` ad alanına taşınmalı, SU24 kontrolleri tamamlanmalıdır [^2]. |
| **S/4HANA Cloud (3SL)** | Fiori Spaces & Pages hiyerarşisi işletilmeden kullanıcı deneyimi sağlanamaz [^6]. | SAP Activate **S4H_1665** standardına uygun `BR_*`, `SP_*`, `PG_*`, `BC_*` sözdizimi kullanılmalıdır [^7]. |
| **Acil Erişim (Firefighter)** | Acil durumlarda verilen süper yetkiler merkezi denetimi delebilir [^3]. | `Z_FF_*` isimlendirmesi, bilet gerekçesi ve Controller onayı işletilmeli, IPS dışı bırakılmalıdır [^3][^10]. |
| **Varsayılan Atanma Değerleri** | SCIM 2.0 SU3 parametrelerini yazamaz; varsayılan tarih `31.12.9999`'dur (`PRGN_CUST`) [^1][^2]. | Veri kısıtları kullanıcı parametresiyle değil, türetilmiş rollerle (Derived Roles) çözülmelidir [^5]. |
| **Çift Yönlü Senkronizasyon** | Çift yönlü kullanıcı yazımı döngüsel çakışmaya (Circular Sync Loop) yol açar [^19]. | Entra ID tekil kimlik kaynağı (SSoT) kalmalı; S/4HANA ve BTP hedef olmalıdır [^8][^10]. |
| **Principal Propagation** | Cloud Connector CA sertifikası STRUST'a yüklenmez ve ICM güven parametreleri girilmezse HTTP 401/403 oluşur [^13]. | STRUST `cAVK` havuzu doldurulmalı; `icm/HTTPS/trust_client_certs = TRUE` yapılandırılmalıdır [^13]. |
| **BTP Yetkilendirme** | BTP'de kullanıcı bazlı manuel rol ataması operasyonel körlük yaratır [^15]. | Entra ID güvenlik grupları doğrudan **BTP Role Collections** ile eşleştirilmelidir (JIT Federation) [^9][^10]. |
| **AI Core & Generative AI** | Yapay zeka modelleri kiracılar arası veri izolasyonu olmadan çalıştırılamaz [^12]. | **Resource Groups** (`resourceGroups`) izolasyonu, Service Key Scopes ve token kotası konulmalıdır [^12]. |
| **SAP Joule & CDS DCL** | Joule verileri klasik transaction'la değil, CDS Views / RAP üzerinden çeker; AUTHORITY-CHECK yetersiz kalır [^16]. | Satır ve alan bazlı veri kısıtlaması **CDS DCL (Data Control Language - Access Controls)** ile sağlanmalıdır [^14][^16]. |
| **User Buffer Senkronizasyonu** | AGR_USERS tarihi dolsa dahi PFUD çalışana kadar `UST04` tampon belleği boşaltılmaz [^17]. | Rol emekliliğinde `PFCG_TIME_DEPENDENCY` (`PFUD` / `RHAUTUSR_COMPARE`) anında tetiklenmelidir [^17]. |
| **Denetim ve SIEM** | Rol değişiklikleri ilişkisel tablolarda gizli kalırsa yetki yükseltme fark edilemez [^1][^11]. | `CDHDR/CDPOS` ve Security Audit Log verileri **Microsoft Sentinel** ile sürekli izlenmelidir [^11]. |
| **Emeklilik Yaşam Döngüsü** | Kullanılmayan roller anında silinirse iş sürekliliği riske girer [^7]. | 90 günlük iç kontrol karantina süresi, `[DEPRECATED]` etiketi ve `SG_SAP_RETIRED_*` filtresi uygulanmalıdır [^8]. |

---

## 10. Resmi Referanslar ve Kaynakça (Official References & Standards)

[^1]: **SAP Learning & SAP Help Portal - ABAP Security Architecture:** *Creating Standard Roles and NetWeaver BC-SEC Core Authorization Concepts (PFCG Fundamentals, Table PRGN_CUST, Field TO_DAT 31.12.9999, SUIM Reports RSUSR100N & RSSCD100_PFCG)*. [https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/creating-standard-roles](https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/creating-standard-roles)
[^2]: **SAP Learning - Exploring the Authorization Concept for SAP S/4HANA and SAP Business Suite:** *Creating Special ABAP Roles and Customizing Authorization Concepts (Customer Name Ranges Z_ and Y_, Namespace Rules, SU24/SU25 Guidelines)*. [https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/creating-special-abap-roles](https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/creating-special-abap-roles)
[^3]: **SAP Help Portal - SAP Governance, Risk, and Compliance (GRC):** *SAP Role Naming Conventions and Emergency Access Management (Firefighter / EAM Architecture, Reason Codes, and Controller Review)*. [https://help.sap.com/docs/SUPPORT_CONTENT/grc/3362386825.html?locale=en-US](https://help.sap.com/docs/SUPPORT_CONTENT/grc/3362386825.html?locale=en-US) / [https://help.sap.com/docs/SUPPORT_CONTENT/grc/3362386889.html?locale=en-US](https://help.sap.com/docs/SUPPORT_CONTENT/grc/3362386889.html?locale=en-US)
[^4]: **SAP Learning - Role Architecture Strategy:** *Implementing a Composite Role Strategy in SAP S/4HANA vs Modern Business Role Combinatorial Explosion in Enterprise Environments*. [https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/implementing-a-composite-role-strategy](https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/implementing-a-composite-role-strategy)
[^5]: **SAP Learning - Derived Roles Architecture:** *Implementing a Derived Role Strategy with Organizational Level Inheritance (`$BUKRS`, `$WERKS`, `$EKORG`) in SAP S/4HANA*. [https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/implementing-a-derived-role-strategy](https://learning.sap.com/courses/exploring-the-authorization-concept-for-sap-s-4hana-and-sap-business-suite/implementing-a-derived-role-strategy)
[^6]: **SAP Community - ERP Architecture Best Practices:** *Naming and Tracking Your Business Roles, Spaces and Pages in the SAP S/4HANA Cloud 3-System Landscape Systems*. [https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/naming-and-tracking-your-business-roles-spaces-and-pages-in-the-sap-s-4hana/ba-p/13561243](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/naming-and-tracking-your-business-roles-spaces-and-pages-in-the-sap-s-4hana/ba-p/13561243)
[^7]: **SAP Activate Methodology Accelerator:** *S4H_1665 How to Set Up a Naming Convention for Business Roles, Spaces and Pages (3-System Landscape)*. [https://support.sap.com/content/dam/SAAP/SAP_Activate/S4H_1665%20How%20to%20Set%20Up%20a%20Naming%20Convention%20for%20Business%20Roles%2c%20Spaces%20and%20Pages%20(3-system%20landscape).pdf](https://support.sap.com/content/dam/SAAP/SAP_Activate/S4H_1665%20How%20to%20Set%20Up%20a%20Naming%20Convention%20for%20Business%20Roles%2c%20Spaces%20and%20Pages%20(3-system%20landscape).pdf)
[^8]: **Microsoft Learn - Microsoft Entra ID Governance:** *Manage Access to Your SAP Applications with Microsoft Entra ID Governance (Lifecycle Workflows, Entitlement Management, Access Packages & Access Reviews)*. [https://learn.microsoft.com/tr-tr/entra/id-governance/sap](https://learn.microsoft.com/tr-tr/entra/id-governance/sap)
[^9]: **Microsoft Learn - Microsoft Entra SaaS Application Integration:** *Configure Single Sign-On and Role Assignment Integration with SAP Business Technology Platform (SAP BTP)*. [https://learn.microsoft.com/tr-tr/entra/identity/saas-apps/sap-hana-cloud-platform-tutorial](https://learn.microsoft.com/tr-tr/entra/identity/saas-apps/sap-hana-cloud-platform-tutorial)
[^10]: **Microsoft Learn & SAP Help Portal - SAP Cloud Identity Services (IPS):** *Configuring SAP BTP and S/4HANA Target Systems with Identity Provisioning (System Properties: abap.role.filter, ips.delete.threshold, and defaultValue Transformations)*. [https://learn.microsoft.com/tr-tr/entra/identity/saas-apps/sap-btp-provisioning-tutorial](https://learn.microsoft.com/tr-tr/entra/identity/saas-apps/sap-btp-provisioning-tutorial) / [https://help.sap.com/docs/identity-provisioning](https://help.sap.com/docs/identity-provisioning)
[^11]: **Microsoft Learn - Microsoft Sentinel Solution for SAP Applications:** *Configure and Prepare Your SAP System for Microsoft Sentinel Continuous Threat Monitoring (Privilege Escalation Detection, Auditing Tables CDHDR, CDPOS, USH02, USH04, and Security Audit Log SM20)*. [https://learn.microsoft.com/en-us/azure/sentinel/sap/preparing-sap](https://learn.microsoft.com/en-us/azure/sentinel/sap/preparing-sap)
[^12]: **SAP Help Portal - SAP AI Core Official Documentation Guide:** *SAP AI Core Architecture, Multi-Tenancy via Resource Groups, Generative AI Hub, Service Keys, Scopes (ai-core.admin, ai-core.write, ai-core.read), and Metering Specifications*. [https://help.sap.com/docs/sap-ai-core](https://help.sap.com/docs/sap-ai-core)
[^13]: **SAP Help Portal - SAP BTP Connectivity Service Official Guide:** *Principal Propagation, SAP Cloud Connector Configuration, Short-Lived X.509 Certificates, STRUST CA Import, ICM Parameters (icm/HTTPS/trust_client_certs), CERTRULE Mapping, and Destination Service Architecture*. [https://help.sap.com/docs/connectivity](https://help.sap.com/docs/connectivity)
[^14]: **SAP Help Portal - SAP Cloud Identity Services - Identity Authentication:** *Identity Directory Architecture, Multi-Tenant Bundles, Real-Time User Provisioning, SAML/OIDC Application Configuration, and User Context Propagation for Joule*. [https://help.sap.com/docs/identity-authentication](https://help.sap.com/docs/identity-authentication)
[^15]: **SAP Help Portal - SAP Business Technology Platform (SAP BTP) Security & Administration Guide:** *Platform Users vs Business Users, Role Collections, Role Templates, Scopes, xs-security.json, XSUAA, SAP AI Launchpad Governance, and Corporate IdP Federation*. [https://help.sap.com/docs/btp](https://help.sap.com/docs/btp)
[^16]: **SAP Help Portal - ABAP Core Data Services (CDS) & RAP Security Guide:** *CDS Data Control Language (DCL), Access Controls, Aspect pfcg_auth, and Row/Field Level Authorization Architecture for OData and SAP Joule*. [https://help.sap.com/docs/SAP_NETWEAVER_750/cc0c305d23dd49f0a44133499443e2e0/ab0c305d23dd49f0a44133499443e2e0.html](https://help.sap.com/docs/SAP_NETWEAVER_750/cc0c305d23dd49f0a44133499443e2e0/ab0c305d23dd49f0a44133499443e2e0.html)
[^17]: **SAP Help Portal - User Administration & Authorization Buffer Reorganization:** *Reorganizing the SAP User Authorization Buffer (Table UST04 / USR04) with Transaction PFUD and Background Program RHAUTUSR_COMPARE (PFCG_TIME_DEPENDENCY)*. [https://help.sap.com/docs/SAP_NETWEAVER_750/5c1839219f354ab99f4414e217c4fa48/526711ec439b11d1896f0000e8322d00.html](https://help.sap.com/docs/SAP_NETWEAVER_750/5c1839219f354ab99f4414e217c4fa48/526711ec439b11d1896f0000e8322d00.html)
[^18]: **IETF SCIM 2.0 Core Schema (RFC 7643):** *System for Cross-domain Identity Management: Core Schema, Enterprise User Extension, and SAP Schema Mapping*. [https://datatracker.ietf.org/doc/html/rfc7643](https://datatracker.ietf.org/doc/html/rfc7643)
[^19]: **IETF SCIM 2.0 Protocol Specification (RFC 7644):** *System for Cross-domain Identity Management: Protocol Specification, Section 3.5.2 (Modifying Resources with PATCH) and Section 3.14 (Concurrency Control using ETags)*. [https://datatracker.ietf.org/doc/html/rfc7644](https://datatracker.ietf.org/doc/html/rfc7644)
[^20]: **IETF RFC 7522:** *Security Assertion Markup Language (SAML) 2.0 Profile for OAuth 2.0 Client Authentication and Authorization Grants*. [https://datatracker.ietf.org/doc/html/rfc7522](https://datatracker.ietf.org/doc/html/rfc7522)
[^21]: **IETF RFC 7523:** *JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants*. [https://datatracker.ietf.org/doc/html/rfc7523](https://datatracker.ietf.org/doc/html/rfc7523)
[^22]: **U.S. Securities and Exchange Commission (SEC) & PCAOB:** *Management's Report on Internal Control Over Financial Reporting and Certification of Disclosure (SOX Section 404 IT General Controls for SAP Systems)*. [https://www.sec.gov/rules/final/33-8238.htm](https://www.sec.gov/rules/final/33-8238.htm)
[^23]: **National Institute of Standards and Technology (NIST):** *NIST Special Publication 800-207: Zero Trust Architecture (Principles of Continuous Verification, Ephemeral Credentials, and Explicit Least Privilege)*. [https://doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207)
[^24]: **International Organization for Standardization (ISO):** *ISO/IEC 27001:2022 Information Security Management Systems - Controls 5.15 (Access Control), 5.18 (Access Rights Lifecycle), and 8.15 (Logging and Monitoring)*. [https://www.iso.org/standard/27001](https://www.iso.org/standard/27001)

---

## 11. Kısaltmalar ve Terimler Sözlüğü (Glossary of Acronyms & Abbreviations)

| Kısaltma | Orijinal Açılımı (İngilizce / Resmi) | Belgedeki Teknik Anlamı ve İşlevi |
|:---|:---|:---|
| **3SL** | 3-System Landscape | 3 Sistemli Ortam. SAP S/4HANA Cloud mimarisinde Geliştirme (DEV), Test (TEST) ve Canlı (PRD) sistem katmanlarından oluşan standart altyapı. |
| **ABAC** | Attribute-Based Access Control | Nitelik Tabanlı Erişim Denetimi. Kullanıcının unvan, departman, şirket kodu gibi dinamik profil niteliklerine göre veri kısıtlaması (CAP CDS katmanı). |
| **ABAP** | Advanced Business Application Programming | SAP'nin kurumsal ERP iş mantığını koşturan ve yetki denetimlerini (`AUTHORITY-CHECK`) yürüten çekirdek programlama dili. |
| **AD** | Active Directory | Şirket içi yerel ağlarda kullanıcı, bilgisayar ve güvenlik gruplarını yöneten Microsoft dizin hizmeti. |
| **API** | Application Programming Interface | Uygulama Programlama Arayüzü. Sistemlerin programatik olarak veri alışverişi yaptığı bağlantı noktası. |
| **BAPI** | Business Application Programming Interface | SAP iş nesnelerine dış sistemlerden programatik erişim sağlayan standart RFC tabanlı metot arabirimi. |
| **BC** | Basis Components | SAP Basis. Sistem altyapısını, veritabanını, sistem parametrelerini ve kullanıcı güvenliğini yöneten temel sistem yönetim katmanı ve modülü. |
| **BDDK** | Bankacılık Düzenleme ve Denetleme Kurumu | Türkiye finans sektöründe bilgi sistemleri güvenliği, yetki yönetişimi ve denetim izi standartlarını belirleyen yasal düzenleyici otorite. |
| **BTP** | SAP Business Technology Platform | SAP'nin bulut entegrasyon, uzantı geliştirme, kimlik federasyonu ve merkezi yetkilendirme platformu. |
| **BW / BW/4HANA** | Business Warehouse / Business Warehouse 4 HANA | SAP'nin kurumsal veri ambarı, analitik modelleme ve raporlama sistemi. |
| **CA** | Certificate Authority | Sertifika Otoritesi. Cloud Connector'da istemci sertifikalarını imzalayan güvenilir kök/ara sertifika makamı. |
| **CAP** | Cloud Application Programming Model | SAP BTP üzerinde kurumsal mikroservisler geliştirmek için kullanılan Java/Node.js mimari çerçevesi. |
| **CDHDR** | Change Document Header | SAP veritabanında kullanıcı, rol ve profil değişikliklerinin başlık bilgilerini (`USERNAME`, `UDATE`, `CHANGENR`) saklayan sistem tablosu. |
| **CDPOS** | Change Document Position | SAP veritabanında hangi tablonun hangi alanının değiştiğini (`TABNAME`, `FNAME`, `VALUE_OLD`, `VALUE_NEW`) tutan değişiklik kalem tablosu. |
| **CDS** | Core Data Services | SAP HANA ve BTP CAP üzerinde veri modellerini ve deklaratif erişim kurallarını (`@restrict`) tanımlayan şema dili. |
| **CERTRULE** | Certificate Rule Mapping | SAP S/4HANA'da X.509 istemci sertifikasındaki alanları SU01 kullanıcı adına dönüştüren standart kural tablosu işlem kodu. |
| **CISO** | Chief Information Security Officer | Kurumsal Bilgi Güvenliği Baş Yetkilisi. Yetki politikalarını, kasa onaylarını ve denetim stratejilerini belirleyen üst yönetici. |
| **CO** | Controlling | SAP Maliyet Muhasebesi ve Kontrol Modülü. Masraf merkezleri bazında yetki kısıtlamalarını yöneten modül. |
| **CUA** | Central User Administration | Merkezi Kullanıcı Yönetimi. Klasik SAP sistemlerinde kullanıcı ve yetkileri tek bir sistemden dağıtan geleneksel RFC tabanlı teknoloji. |
| **DCL** | Data Control Language | Veri Denetim Dili. SAP ABAP Core Data Services (CDS) mimarisinde satır ve alan bazlı veri erişim denetimi tanımlayan yetki şeması (`aspect pfcg_auth`). |
| **EAM** | Emergency Access Management | Acil Erişim Yönetimi. SAP GRC modülünde süper yönetici yetkilerinin süreli, kayıtlı ve onaylı olarak verilmesini sağlayan yangın söndürücü (Firefighter) mekanizması. |
| **ESS** | Employee Self-Service | Çalışan Öz-Servisi. Tüm şirket personelinin izin talebi, bordro görüntüleme gibi temel işlemleri yaptığı Fiori portal arayüzü. |
| **EXTID** | External Identity | Dış Kimlik. SAP sisteminde kullanıcının e-posta, UPN veya X.509 subject adı ile eşleştirildiği dış kimlik kaydı (`USREXTID`). |
| **FFID** | Firefighter Identity | Acil Durum Kullanıcı Kimliği. Kritik arızalarda süreli olarak devralınan yüksek yetkili sistem kimliği. |
| **FI** | Financial Accounting | SAP Finansal Muhasebe Modülü. Genel muhasebe ve şirket kodu kısıtlı rolleri barındıran çekirdek ERP modülü. |
| **FUE** | Full User Equivalent | Tam Kullanıcı Eşdeğeri. SAP bulut lisanslamasında farklı kullanıcı tiplerini tek bir havuzda ağırlıklandıran lisans birimi. |
| **GDPR** | General Data Protection Regulation | Avrupa Birliği Genel Veri Koruma Tüzüğü. Kimlik verilerinin işlenmesi ve aktarılmasında uyulması gereken yasal çerçeve. |
| **GRC** | Governance, Risk, and Compliance | Kurumsal Yönetişim, Risk ve Uyumluluk. Görevler Ayrılığı (SoD) çatışmalarını ve acil durum erişimlerini yöneten SAP yazılım paketi. |
| **HCM** | Human Capital Management | SAP İnsan Sermayesi Yönetimi. Personel ana verisi ve organizasyonel hiyerarşiyi yöneten sistem. |
| **HR** | Human Resources | İnsan Kaynakları. Kimlik yaşam döngüsünün tetikleyicisi olan kurumsal birim. |
| **HSM** | Hardware Security Module | Donanımsal Güvenlik Modülü. Kriptografik anahtarları ve servis şifrelerini fiziksel olarak koruyan yüksek güvenlikli kasa cihazı. |
| **HTTP / HTTPS** | Hypertext Transfer Protocol (Secure) | Web üzerinde güvenli veri iletimi sağlayan çekirdek internet iletişim protokolü. |
| **IAG** | SAP Cloud Identity Access Governance | SAP'nin bulut tabanlı Görevler Ayrılığı (SoD), erişim talebi ve yetki sertifikasyonu yönetim çözümü. |
| **IAM** | Identity and Access Management | Kimlik ve Erişim Yönetimi. Doğru kişilerin doğru kaynaklara doğru zamanda ve doğru nedenlerle erişmesini sağlayan kurumsal disiplin. |
| **IAS** | SAP Cloud Identity Services - Identity Authentication | SAP bulut ekosisteminin kurumsal kimlik doğrulama, tekil oturum açma (SSO) ve federasyon proxy servisi. |
| **ICM** | Internet Communication Manager | İnternet İletişim Yöneticisi. SAP NetWeaver altyapısında HTTP/HTTPS isteklerini karşılayan çekirdek ağ katmanı (`icm/HTTPS/trust_client_certs`). |
| **IdP** | Identity Provider | Kimlik Sağlayıcı. Kullanıcıların kimliğini doğrulayan ve doğrulanmış dijital biletler (token) üreten merkezi güven makamı (Microsoft Entra ID). |
| **IETF** | Internet Engineering Task Force | İnternet Mühendisliği Görev Gücü. SCIM, HTTP, TLS gibi küresel internet protokol ve standartlarını yayımlayan uluslararası kuruluş. |
| **IGA** | Identity Governance and Administration | Kimlik Yönetişimi ve İdare. Rol yaşam döngüsü, yetki sertifikasyonu ve erişim paketlerini yöneten üst seviye kurumsal disiplin. |
| **IPS** | SAP Cloud Identity Services - Identity Provisioning | Kimlik ve yetkileri kaynak sistemlerden okuyup hedef sistemlere SCIM protokolüyle dağıtan bulut ETL provizyon motoru. |
| **ISO** | International Organization for Standardization | Uluslararası Standardizasyon Kuruluşu. Bilgi güvenliği standartlarını (ISO/IEC 27001) yayımlayan küresel otorite. |
| **IT** | Information Technology | Bilgi Teknolojileri. Sistem altyapısı, dizin hizmetleri ve yetkilendirme süreçlerini işleten teknik birim. |
| **JIT** | Just-In-Time Provisioning | Tam Zamanında Provizyon. Kullanıcının oturum açtığı anda dizin biletindeki niteliklerle sistemde anlık oluşturulması veya yetkilendirilmesi. |
| **JML** | Joiner, Mover, Leaver | İşe Giriş (Joiner), Görev Değişimi (Mover) ve İşten Ayrılış (Leaver) adımlarını kapsayan uçtan uca çalışan kimlik yaşam döngüsü modeli. |
| **JSON** | JavaScript Object Notation | SCIM REST API istek ve yanıtlarında kullanılan, insan tarafından okunabilir hafif veri değişim formatı. |
| **JWT** | JSON Web Token | Taraflar arasında güvenli bilgi aktarımı sağlayan, dijital olarak imzalanmış kompakt JSON bilet standardı (RFC 7519). |
| **KBA** | Knowledge Base Article | SAP Destek Bilgi Bankası Makalesi. SAP ürünlerindeki teknik konfigürasyon ve hata giderme kılavuzları. |
| **KVKK** | Kişisel Verilerin Korunması Kanunu | 6698 Sayılı Kişisel Verilerin Korunması Kanunu. Türkiye'de kimlik verilerinin işlenmesi ve saklanmasını denetleyen mevzuat. |
| **MFA** | Multi-Factor Authentication | Çok Faktörlü Doğrulama. Kullanıcının parola haricinde ek bir faktörle kimliğini kanıtlamasını zorunlu kılan güvenlik mekanizması. |
| **MM** | Materials Management | SAP Malzeme Yönetimi ve Satınalma Modülü. Satınalma siparişleri ve stok yönetimini içeren çekirdek ERP modülü. |
| **NIST** | National Institute of Standards and Technology | ABD Ulusal Standartlar ve Teknoloji Enstitüsü. Sıfır Güven (Zero Trust - SP 800-207) mimarisinin küresel referans otoritesi. |
| **OData** | Open Data Protocol | REST tabanlı web servisleri üzerinden SAP verilerine erişim ve CRUD operasyonları sağlayan açık veri protokolü. |
| **OIDC** | OpenID Connect | OAuth 2.0 üzerine inşa edilmiş, JWT tabanlı modern bulut kimlik doğrulama protokolü. |
| **PAM** | Privileged Access Management | Ayrıcalıklı Erişim Yönetimi. Sistem yöneticisi ve kritik yetkilerin süreli ve kasadan çekilerek verilmesini yöneten güvenlik disiplini. |
| **PCE** | Private Cloud Edition | SAP S/4HANA Cloud Özel Bulut Sürümü. Müşteriye özel ayrılmış bulut altyapısında çalışan S/4HANA ERP sistemi. |
| **PFCG** | Profile Generator | SAP ABAP sisteminde yetki rollerinin, işlem kodlarının ve yetki nesnelerinin tasarlandığı temel yönetim işlem kodu. |
| **PFUD** | Profile Generator User Buffer Sync | SAP NetWeaver'da rol geçerlilik tarihlerini kullanıcı yetki tampon belleğine (`UST04`) eşitleyen standart toplu işlem programı (`RHAUTUSR_COMPARE`). |
| **PM** | Plant Maintenance | SAP Bakım ve Onarım Modülü. Tesis ekipmanları ve bakım iş emirlerini yöneten modül. |
| **PP** | Production Planning | SAP Üretim Planlama Modülü. Üretim emirleri ve kapasite planlama yetkilerini barındıran modül. |
| **QA** | Quality Assurance | Kalite ve Test Ortamı. Yetki değişikliklerinin ve mock simülasyonların canlıya geçmeden önce doğrulandığı sistem katmanı. |
| **RACI** | Responsible, Accountable, Consulted, Informed | Sorumlu, Hesap Verebilir, Danışılan, Bilgilendirilen. Rol ve yetki süreçlerindeki kurumsal görev paylaşım matrisi. |
| **RAP** | RESTful Application Programming Model | SAP S/4HANA ve ABAP Cloud üzerinde OData servisleri ve bulut yerel iş uygulamaları geliştirmek için kullanılan modern programlama modeli. |
| **RBAC** | Role-Based Access Control | Rol Tabanlı Erişim Denetimi. Yetkilerin doğrudan kullanıcılara değil, iş fonksiyonlarını temsil eden rollere atanması güvenlik mimarisi. |
| **RFC** | Request for Comments / Remote Function Call | 1. IETF internet standart belgesi (Örn: RFC 7643).<br>2. SAP sistemleri arasında fonksiyon çağrısı sağlayan ikili iletişim protokolü. |
| **S/4HANA** | SAP Business Suite 4 SAP HANA | SAP'nin yüksek hızlı bellek içi HANA veritabanı üzerinde koşan yeni nesil ERP kurumsal yazılım paketi. |
| **SAC** | SAP Analytics Cloud | SAP'nin BTP üzerinde çalışan bulut tabanlı iş zekası, analitik ve planlama platformu. |
| **SAL** | Security Audit Log | SAP Güvenlik Denetim Günlüğü. İşlem kodları, oturum açma denemeleri ve yetki hatalarının (`SM20`) kayıt defteri. |
| **SAML** | Security Assertion Markup Language | XML tabanlı, kurumlar arası tekil oturum açma (SSO) ve yetki federasyonu sağlayan açık güvenlik standardı (SAML 2.0). |
| **SCC** | SAP Cloud Connector | Bulut uygulamaları ile şirket içi SAP sistemleri arasında şifreli ters tünel kuran bağlantı aracısı. |
| **SCIM** | System for Cross-domain Identity Management | IETF RFC 7643/7644 standartlarına dayalı, REST/JSON tabanlı bulut kimlik provizyonu ve şema yönetim protokolü. |
| **SD** | Sales and Distribution | SAP Satış ve Dağıtım Modülü. Müşteri siparişleri ve faturalama yetkilerini içeren çekirdek modül. |
| **SF** | SAP SuccessFactors | Çalışan ana verisinin ve pozisyon kodlarının tutulduğu tekil güven kaynağı (SSoT) bulut İK sistemi. |
| **SIEM** | Security Information and Event Management | Güvenlik Bilgi ve Olay Yönetimi. Denetim loglarının toplandığı kurumsal izleme platformu (Splunk, Microsoft Sentinel). |
| **SoD** | Segregation of Duties | Görevler Ayrılığı. Suistimalleri önlemek amacıyla birbirini denetleyen işlemlerin farklı kullanıcılara verilmesi kuralı. |
| **SOX** | Sarbanes-Oxley Act of 2002 | ABD Şirketler Muhasebe Reformu Kanunu. Bölüm 404 uyarınca IT yetki denetimlerini zorunlu kılan yasal düzenleme. |
| **SSoT** | Single Source of Truth | Tekil Güven Kaynağı. Kurumsal kimlik verisinin yalnızca tek bir ana sistemden doğması ilkesi. |
| **STRUST** | Trust Manager | SAP NetWeaver ve S/4HANA'da SSL/TLS sertifikalarını, istemci sertifika otoritelerini (`cAVK`) ve şifreleme anahtarlarını yöneten güven işlem kodu. |
| **SUIM** | User Information System | SAP Kullanıcı Bilgi Sistemi. Kullanıcı, rol ve profil değişiklik belgelerinin sorgulandığı denetim raporları bütünü (`RSUSR100N`). |
| **T-Code** | Transaction Code | SAP İşlem Kodu. SAP GUI veya arayüzünde belirli bir uygulamayı başlatan kısa komut kodu (`PFCG`, `SU01`, `SUIM`). |
| **UI** | User Interface | Kullanıcı Arayüzü. Kullanıcıların sistemle etkileşime girdiği görsel katman (SAP GUI, Fiori Launchpad). |
| **UPN** | User Principal Name | Active Directory ve Microsoft Entra ID'de kullanıcının e-posta biçimindeki benzersiz oturum açma adı. |
| **UST04** | User Buffer Table | SAP veritabanında kullanıcılara atanmış yetki profillerinin önbelleğini saklayan tampon bellek tablosu (`PFUD` ile güncellenir). |
| **UUID** | Universally Unique Identifier | Evrensel Benzersiz Kimlik. Sistemler arası eşleştirmede kullanılan 128-bitlik benzersiz kimlik dizgisi (`userUuid`). |
| **X.509** | X.509 Public Key Certificate | Açık anahtarlı şifrelemede dijital kimlik kanıtı sunan standart sertifika formatı. Cloud Connector Principal Propagation'da kullanılır. |
| **XSUAA** | Extended Services User Account and Authentication | SAP BTP Cloud Foundry ortamında OAuth 2.0 token üreten, Role Collection ve scope atamalarını yöneten çekirdek yetkilendirme servisi. |
