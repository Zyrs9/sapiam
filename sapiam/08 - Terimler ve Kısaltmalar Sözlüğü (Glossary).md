---
title: Terimler ve Kısaltmalar Sözlüğü (Glossary & Acronyms)
tags:
  - glossary
  - acronyms
  - terminology
  - reference
  - standards
status: approved
version: "3.2"
updated: 2026-10-22
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Terimler ve Kısaltmalar Sözlüğü (Glossary & Acronyms)

Bu belge, SAP ekosisteminde kimlik yönetimi (IAM), yaşam döngüsü otomasyonu (JML) ve bulut yetkilendirme projelerinde kullanılan tüm teknik kavram, protokol ve kısaltmaların standart kurumsal sözlüğüdür.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Kurumsal Strateji: [[01 - Kurumsal Strateji, Olgunluk ve ROI]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* JSON Kodları: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Güvenlik & Kasa: [[05 - Güvenlik, Denetim ve Break-Glass]]
* Kurulum Rehberi: [[06 - Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi]]
* Taktiksel Saha Rehberi: [[07 - Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi]]
* Rol Standartları & İleri IAM: [[09 - Rol Standartları, Yaşam Döngüsü ve İleri Seviye IAM Araştırmaları]]

---

## 1. Kısaltmalar Sözlüğü (Acronyms & Abbreviations)

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
| **FUE** | Full User Equivalent | Tam Kullanıcı Eşdeğeri. SAP bulut lisanslamasında farklı kullanıcı tiplerini (Professional, Functional, Productivity) tek bir katsayı havuzunda toplayan metrik birim. |
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

## 2. İçerik ve Kavramsal Terimler Sözlüğü (Glossary of Core Concepts)

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
