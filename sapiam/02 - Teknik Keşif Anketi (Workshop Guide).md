---
title: Teknik Keşif Anketi (Workshop Guide)
tags:
  - questionnaire
  - workshop
  - requirements
  - basis
  - hr
  - active-directory
status: approved
version: "3.0"
updated: 2026-10-19
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Teknik Keşif Anketi (Workshop Guide)

Bu belge, paydaş birimlerle (**İK/SuccessFactors**, **IT Altyapı/Dizin Hizmetleri**, **SAP Basis/Yetki** ve **Siber Güvenlik**) yürütülecek mimari keşif çalıştaylarında (discovery workshops) sistem gereksinimlerini ve saha parametrelerini netleştirmek için hazırlanmış resmi soru envanteridir.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]

---

## 1. Paydaş Keşif Matrisi

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

---

## BÖLÜM 1: İK ve SAP SuccessFactors Ekibi (Source of Truth)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **SF-1** | Çalışan tekil anahtarı (Correlation Key) olarak hangi alan kullanılmaktadır (`personIdExternal`, `userId`, `nationalId`)? | Uçtan uca sistemler arası (AD `employeeID`, IAS `userUuid`, S/4HANA `User Alias`) eşleştirmenin temelini oluşturur. Değişmez (immutable) olmalıdır. | RFC 7643 SCIM Core Schema [^1] | Tekil alan adı ve veri tipi (Örn: 8 haneli numerik string). |
| **SF-2** | Gelecek tarihli işe alımlar (Future Hires) için süreç nasıldır? Day-1 erişimi için önceden provizyon (pre-provisioning) isteniyor mu? | Çalışanın ilk iş gününde oturum açabilmesi için AD ve SAP hesaplarının sözleşme tarihinden önce (T-3 gün) açılıp kilitli bekletilmesi gerekir. | SAP SuccessFactors Employee Central Master Data Guide [^2] | Ön-provizyon süresi (Örn: T-3 gün oluştur, T-0 aktif et). |
| **SF-3** | Departman/Pozisyon değişimleri (Mover) SF üzerinde hangi `Event Reason` kodları ile tetiklenmektedir? | Mover senaryolarında eski yetkilerin alınıp yenilerinin tanımlanması için delta sync filtrelerinin belirlenmesi gerekir. | SAP Help: Managing Employment Lifecycle Events [^3] | Mover tetikleyici event kodları listesi (Örn: `JOB_CHANGE`, `PROMOTION`). |
| **SF-4** | İşten ayrılış (Termination / Leaver) işlemlerinde saat kuralı ve Acil Çıkış (Immediate Revocation) politikası nedir? | Standart çıkışlarda effective date gece yarısı mı devreye giriyor? Güvenlik riski durumunda anında hesap kilitlenmesi için bypass akışı var mı? | NIST SP 800-207 Control Tenet 5 [^4] | Standart çıkış saati ve acil çıkış SLA'i (Örn: Max 15 dk). |
| **SF-5** | Dış kaynak (Contractor / Vendor / Stajyer) kimlikleri SuccessFactors'te mi tutuluyor yoksa doğrudan AD'de mi açılıyor? | Eğer dış kaynaklar SF'te değilse, AD'deki hesaplarının yaşam döngüsü için ayrı bir IAM kural seti kurulmalıdır. | SAP SuccessFactors Contingent Workforce Guide [^5] | Dış kaynakların ana veri kaynağı (SF vs Doğrudan AD/Entra). |
| **SF-6** | SF ile Active Directory arasındaki mevcut entegrasyon yöntemi nedir? | Entra Inbound Provisioning API mi, özel script mi, yoksa manuel CSV export mu yapılıyor? | Microsoft Entra Inbound Provisioning with SuccessFactors [^6] | Entegrasyon tipi, frekansı (Örn: Her 30 dk). |
| **SF-7** | **SuccessFactors Onboarding 2.0 modülü kullanılıyor mu? Pre-hire kullanıcıları işe başlamadan önce IAS üzerinden mi oturum açıyor?** | **13 Kasım 2026 SAP IAS zorunluluğu kapsamında Onboarding 2.0 dış aday girişleri ilk etkilenecek alandır. Adayların henüz AD hesabı yokken IAS yerel kimliğiyle giriş yapması gerekebilir.** | **SAP KBA 2791410 & SAP Note 3048590** [^7] | **Onboarding 2.0 kullanımı (Evet/Hayır) ve aday giriş yöntemi.** |

---

## BÖLÜM 2: IT Altyapı ve Dizin Hizmetleri (Active Directory / Microsoft Entra ID)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **AD-1** | Dizin mimariniz nedir (On-Premises AD, Hibrit Entra ID Connect / Cloud Sync, Pure Entra ID)? | IPS, doğrudan Entra ID Graph API (SCIM) üzerinden mi yoksa On-Prem AD üzerinden Cloud Connector / LDAP ile mi okuma yapacak? | Microsoft Graph API Users & Groups [^8] | Dizin topolojisi ve yetkili tenant/domain bilgisi. |
| **AD-2** | SF'ten gelen çalışan kimlik numarası AD'de hangi attribute'a map edilmektedir? | Çoğu kurumda `employeeID` veya `employeeNumber`, bazen de `extensionAttribute1-15` kullanılır. IPS transformation'da `sourcePath` olacaktır. | RFC 7643 Section 4.1 Enterprise User Extension [^1] | Örn: `$.employeeId` veya `$.extensionAttribute3`. |
| **AD-3** | Kullanıcı adı standardı ve UPN/E-posta ilişkisi nasıldır? | `sAMAccountName` mı `userPrincipalName` (UPN) mı esas alınacak? UPN ile `mail` her zaman birebir eşit midir? | Microsoft Learn: User Name Policies | UPN formatı ve istisna durumları. |
| **AD-4** | SAP Güvenlik Grupları için isimlendirme standardı (Naming Convention) belirlendi mi? | IPS'in AD'deki 10.000 grubu çekmesini engellemek için sadece SAP gruplarını regex ile filtrelemeliyiz. | SAP Help: Group Filtering in IPS [^9] | Örn: `SG_SAP_<SISTEM>_<ROL>_<VERI_ALANI>`. |
| **AD-5** | İç içe gruplar (Nested Groups) kullanılacak mı? Graph API rate-limit ve throttling önlemi alındı mı? | Entra ID / AD tarafında iç içe grupların çözülmesi (`expandNestedGroups`) ve filtresiz sorgular Microsoft Graph API rate-limit/throttling sınırlarına takılarak IPS senkronizasyonunu yarıda kesebilir. Sunucu taraflı filtre (`displayName startswith 'SG_SAP_'`) ve `ips.job.bulk.page.size = 100` şarttır. | Microsoft Graph API Throttling Guidance & SAP Help [^8] [^9] | Nested group kullanımı, derinlik sınırı ve filtre onay durumu. |
| **AD-6** | Bir çalışan işten ayrıldığında AD'de uygulanan standart politika nedir? | `accountEnabled = false` mı yapılıyor, kullanıcı adı önüne `DISABLED_` mı ekleniyor, gruplardan anında çıkarılıyor mu? | Microsoft Entra De-provisioning Policy | Leaver esnasında AD kullanıcı durumu ve grup aksiyonları. |
| **AD-7** | **Microsoft Entra ID tarafında P1 veya P2 lisansı var mı? "Dynamic User Groups" (Dinamik Gruplar) özelliği kullanılabilir mi?** | **P1/P2 varsa unvan, departman ve şirket koduna göre AD grupları kural bazlı dinamik dolar; IT biriminin grup ekleme/çıkarma için manuel iş yapması veya script yazması gerekmez.** | **Microsoft Learn: Dynamic Membership Rules** [^10] | **Entra ID Lisans Seviyesi (Free, P1, P2) ve dinamik grup onayı.** |

---

## BÖLÜM 3: SAP Basis & Güvenlik/Yetki Ekibi (S/4HANA, BTP, SAC)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **BASIS-1** | S/4HANA dağıtım modeli ve versiyonu nedir (Cloud Public, Private Cloud Edition, On-Premise 2022/2023)? | Dağıtım modeline göre provizyon protokolü değişir: Public Cloud ise SCIM 2.0; On-Prem/PCE ise Cloud Connector üzerinden SCIM API (`/sap/bc/scim`) veya SAP AS ABAP RFC. | SAP Help: SAP S/4HANA Target System in IPS [^11] | S/4HANA sürümü, NetWeaver platform seviyesi. |
| **BASIS-2** | S/4HANA sistemlerinde SAP Cloud Connector (SCC) kurulu ve yapılandırılmış mı? | BTP ve IPS'in on-premise ağdaki S/4HANA'ya güvenli erişebilmesi için SCC ters-tüneli şarttır. | SAP Help: Cloud Connector Overview [^12] | SCC erişilebilirliği ve subaccount bağ durumu. |
| **BASIS-3** | Principal Propagation için Kurumsal CA (X.509 Kısa Ömürlü Sertifika) altyapısı hazır mı? | Kullanıcının BTP veya Fiori Launchpad üzerinden S/4HANA'ya geçerken arka planda kendi kimliğiyle işlem yapması için X.509 istemci sertifikası üretilmesi gerekir. | SAP Help: Configuring Principal Propagation [^13] | SCC CA kurulum ve trust ilişkisi durumu. |
| **BASIS-4** | S/4HANA'da Business Partner (BP) ve Kullanıcı (SU01) senkronizasyonu nasıl yürütülmektedir? | S/4HANA'da her çalışan bir Business Partner'dır (CVI). IPS kullanıcıyı SU01'e açtığında BP eşleştirmesi otomatik mi yapılıyor yoksa HCM Employee Sync mi çalışıyor? | SAP Note 2570961: Customer Vendor Integration (CVI) [^14] | BP-User entegrasyon yöntemi (CVI senkronizasyonu). |
| **BASIS-5** | SAP BTP Subaccount mimarisi ve Global Account yapısı nasıldır? | Kaç adet subaccount var (DEV, QA, PRD)? Subaccount seviyesinde Trust Configuration'da doğrudan Entra ID mi bağlı, yoksa merkezi SAP IAS OIDC Proxy mi kullanılıyor? | SAP Help: Setting Up Trust in BTP [^15] | BTP Subaccount listesi ve mevcut Trust yapılandırması. |
| **BASIS-6** | SAP Analytics Cloud (SAC) için SCIM lisansı/API erişimi aktif mi? | SAC üzerinde kullanıcı ve Team yönetimi için SAC Custom SAML IdP ve SCIM API yetkisi olan OAuth Client hazır mı? | SAP Help: Managing Users in SAP Analytics Cloud [^16] | SAC API erişim durumu ve lisans tipi. |
| **BASIS-7** | **Kurumunuza ait kaç adet SAP Cloud Identity Services (IAS) tenant'ı mevcut?** | **SAP her müşteriye lisans kapsamında 1 Test + 1 Production IAS tenant'ı sağlar. Test ve QA ortamlarının canlıyı etkilemeden doğrulanabilmesi için ikinci tenant'ın aktif olması şarttır.** | **SAP Note 2753435: Tenant Model for IAS** [^17] | **Mevcut IAS tenant sayısı ve URL'leri.** |
| **BASIS-8** | **S/4HANA'da Firefighter (Geçici Acil Erişim) hesapları ve manuel atanan istisnai teknik roller nasıl korunacak?** | **IPS authoritative çalıştığında acil durum rollerini SU01'den silebilir; bu rollerin IPS kapsamı dışında tutulması (exclusion filter) için prefix standardı nedir?** | **SAP Help: IPS Transformation Conditions** [^9] | **Firefighter rol/kullanıcı formatı (Örn: `Z_FF_*`).** |
| **BASIS-9** | **S/4HANA Yetki Tasarımı (PFCG) ve Rol İsimlendirme Matrisi teslim süreci belirlendi mi?** | **Yetki içeriği (kilit) S/4HANA PFCG içinde modül ekiplerince tasarlanır; IPS bu rolleri AD grupları üzerinden dağıtır (anahtar teslimi). Modül danışmanlarının tasarladıkları standart teknik rolleri (Örn: `Z_MM_BUYER_1010`) merkezi matrise teslim formatı ve SLA'i netleştirilmelidir.** | SAP Help: PFCG Role Administration [^6] | **Teknik rol envanter listesi ve Rol Eşleştirme Matrisi mutabakatı.** |
| **BASIS-10** | **Mevcut (Brownfield) SU01 kullanıcılarının `externalId` / Alias alanlarında personel numarası eşitlemesi yapıldı mı?** | **Mevcut S/4HANA sisteminde önceden elle açılmış kullanıcıların `SU01` Alias / `externalId` alanlarında İK personel numarası (`personIdExternal` / `employeeID`) bulunmazsa; IPS bu kullanıcıları tanıyamaz, aynı `userName` ile yeni kayıt açmaya çalışırken çakışma yaşanır ve provizyon hata verir.** | **SAP Note 2626880: SCIM User Provisioning Conflict Handling** | **Mevcut kullanıcıların Alias/externalId temizlik durumu (Hazır / Planlanacak).** |

---

## BÖLÜM 4: BTP Mimari ve Bilgi Güvenliği (Architecture & Cyber Security)

| No | Soru / Konu | Teknik Gerekçe & Mimari Etki | Resmi Referans / Standart | Beklenen Yanıt / Çıktı |
|:---|:---|:---|:---|:---|
| **BTP-1** | **BTP üzerinde geliştirilen uygulamalarda yetkilendirme XSUAA (Role Collection) ile mi yoksa yeni nesil AMS (Authorization Management Service) ile mi yönetilecek?** | **CAP Java/Node.js projelerinde ve BTP Cloud Foundry/Kyma ortamlarında XSUAA standarttır; ancak yeni mimarilerde AMS kullanımı BTP token yapısını ve rol hiyerarşisini doğrudan etkiler.** | **SAP Help: Authorization and Trust Management Service** [^18] | **Tercih edilen yetki motoru (XSUAA vs AMS).** |
| **SEC-1** | **MFA (Çok Faktörlü Doğrulama) tetiklemesi nerede yapılacak (Entra ID Conditional Access mi, yoksa SAP IAS Risk-Based Authentication mı)?** | **Her iki tarafta da MFA açılırsa çalışanlar çift faktör döngüsüne (Double-MFA) girerek kilitlenir. Mimari ilke olarak MFA kurumsal IdP olan Entra ID'de işletilmeli; IAS pass-through/proxy çalışmalıdır.** | **NIST SP 800-63B: Digital Identity Guidelines** [^19] | **MFA politikası ve yetkili karar mercii.** |
| **SEC-2** | **Görevler Ayrılığı (SoD) ve Yetki Çakışması denetimi nasıl sağlanacak? SAP IAG (Cloud Identity Access Governance) entegrasyonu planlanıyor mu?** | **IPS körlemesine grup basarsa finansal denetimlerde (SOX/BDDK) "Satıcı Açan Kişinin Fatura Onaylaması" gibi SoD açıkları oluşur. Kritik roller için IPS öncesi bir onay/GRC katmanı gerekip gerekmediği netleşmelidir.** | **SAP Help: SAP Cloud Identity Access Governance** [^20] | **SoD risk kontrol mekanizması (Manuel vs SAP IAG/GRC).** |

---

## Resmi Referanslar ve Kaynakça

[^1]: **IETF RFC 7643:** *System for Cross-domain Identity Management: Core Schema*. [https://datatracker.ietf.org/doc/html/rfc7643](https://datatracker.ietf.org/doc/html/rfc7643)
[^2]: **SAP Help Portal:** *Employee Central Master Data Guide*. [https://help.sap.com/docs/SAP_SUCCESSFACTORS_EMPLOYEE_CENTRAL](https://help.sap.com/docs/SAP_SUCCESSFACTORS_EMPLOYEE_CENTRAL)
[^4]: **NIST SP 800-207:** *Zero Trust Architecture*. [https://doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207)
[^6]: **Microsoft Learn:** *Plan cloud HR application to Microsoft Entra user provisioning*. [https://learn.microsoft.com/en-us/entra/identity/app-provisioning/plan-cloud-hr-provision](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/plan-cloud-hr-provision)
[^8]: **Microsoft Learn:** *Microsoft Graph REST API v1.0 reference*. [https://learn.microsoft.com/en-us/graph/api/overview](https://learn.microsoft.com/en-us/graph/api/overview)
[^9]: **SAP Help Portal:** *Identity Provisioning - List of Properties & Transformation Operations*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties](https://help.sap.com/docs/identity-provisioning/identity-provisioning/list-of-properties)
[^10]: **Microsoft Learn:** *Rules for dynamically populated groups membership in Microsoft Entra ID*. [https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
[^11]: **SAP Help Portal:** *SAP S/4HANA On-Premise and Private Cloud as Target System in IPS*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-s-4hana](https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-s-4hana)
[^12]: **SAP Help Portal:** *SAP BTP Connectivity - Cloud Connector*. [https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/cloud-connector](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/cloud-connector)
[^13]: **SAP Help Portal:** *Configuring Principal Propagation to an ABAP System for HTTPS*. [https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/configuring-principal-propagation-to-abap-system-for-https](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/configuring-principal-propagation-to-abap-system-for-https)
[^14]: **SAP Note 2570961:** *Simplification Item: Customer-Vendor Integration (CVI) in SAP S/4HANA*. [https://me.sap.com/notes/2570961](https://me.sap.com/notes/2570961)
[^15]: **SAP Help Portal:** *Establish Trust and Federation Between UAA and Identity Authentication*. [https://help.sap.com/docs/btp/sap-business-technology-platform/establish-trust-and-federation-between-uaa-and-identity-authentication](https://help.sap.com/docs/btp/sap-business-technology-platform/establish-trust-and-federation-between-uaa-and-identity-authentication)
[^16]: **SAP Help Portal:** *Enabling a Custom SAML Identity Provider in SAP Analytics Cloud*. [https://help.sap.com/docs/SAP_ANALYTICS_CLOUD/00f68c2e08b941f081002fd3691d86a7/e4284d72d2444cfa88eb2203063ea5a3.html](https://help.sap.com/docs/SAP_ANALYTICS_CLOUD/00f68c2e08b941f081002fd3691d86a7/e4284d72d2444cfa88eb2203063ea5a3.html)
[^17]: **SAP Note 2753435:** *Identity Authentication Tenant Model and Test Tenant Requests*. [https://me.sap.com/notes/2753435](https://me.sap.com/notes/2753435)
[^18]: **SAP Help Portal:** *What Is SAP Authorization and Trust Management Service (XSUAA)?*. [https://help.sap.com/docs/btp/sap-business-technology-platform/what-is-sap-authorization-and-trust-management-service](https://help.sap.com/docs/btp/sap-business-technology-platform/what-is-sap-authorization-and-trust-management-service)
[^19]: **NIST SP 800-63B:** *Digital Identity Guidelines - Authentication and Lifecycle Management*. [https://doi.org/10.6028/NIST.SP.800-63b](https://doi.org/10.6028/NIST.SP.800-63b)
[^20]: **SAP Help Portal:** *SAP Cloud Identity Access Governance (IAG) Overview*. [https://help.sap.com/docs/SAP_CLOUD_IDENTITY_ACCESS_GOVERNANCE](https://help.sap.com/docs/SAP_CLOUD_IDENTITY_ACCESS_GOVERNANCE)
