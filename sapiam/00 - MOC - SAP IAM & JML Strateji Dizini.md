---
title: SAP IAM & JML Kurumsal Mimari Haritası (MOC)
tags:
  - sap/iam
  - sap/btp
  - architecture
  - moc
  - zero-trust
status: approved
version: "3.0"
updated: 2026-10-19
---

# SAP IAM & JML Kurumsal Mimari Haritası (Map of Content)

Bu dijital bilgi kasası (Vault); **SAP SuccessFactors $\rightarrow$ Microsoft Entra ID $\rightarrow$ SAP Cloud Identity Services (IAS/IPS) $\rightarrow$ SAP BTP & S/4HANA** uçtan uca kimlik ve erişim yaşam döngüsü otomasyon projesinin teknik, mimari ve stratejik tasarım belgelerini içerir.

Tüm teknik ifadeler, mimari desenler ve konfigürasyon kuralları **SAP resmi dokümantasyonu (SAP Help Portal, SAP Notes/KBAs)**, **IETF RFC standartları** ve **NIST SP 800-207** çerçevesine dayandırılmıştır.

---

## Not Haritası ve Modüler Navigasyon

Obsidian Graph View üzerinde birbirine bağlı olan 8 temel mimari modül aşağıda tematik sütunlarla modellenmiştir:

```
                                  ┌────────────────────────────────────────────────────────┐
                                  │      [[00 - MOC - SAP IAM & JML Strateji Dizini]]      │
                                  └──────────────────────────┬─────────────────────────────┘
                                                             │
                ┌────────────────────────────┬───────────────┴───────────────┬────────────────────────────┐
                ▼                            ▼                               ▼                            ▼
     ┌──────────────────────┐     ┌──────────────────────┐        ┌──────────────────────┐     ┌──────────────────────┐
     │   STRATEJİ & KEŞİF   │     │   MİMARİ & TAKTİK    │        │  UYGULAMA & KODLAR   │     │  GÜVENLİK & SÖZLÜK   │
     ├──────────────────────┤     ├──────────────────────┤        ├──────────────────────┤     ├──────────────────────┤
     │ • [[01 - Kurumsal    │     │ • [[03 - Çekirdek    │        │ • [[04 - SAP IPS     │     │ • [[05 - Güvenlik,   │
     │   Strateji & ROI]]   │     │   Mimari & JML]]     │        │   JSON Kod Deposu]]  │     │   Denetim & Kasa]]   │
     │ • [[02 - Teknik      │     │ • [[07 - Taktiksel   │        │ • [[06 - Uçtan Uca   │     │ • [[08 - Terimler ve │
     │   Keşif Anketi]]     │     │   Operasyon]]        │        │   Kurulum Rehberi]]  │     │   Kısaltmalar]]      │
     └──────────────────────┘     └──────────────────────┘        └──────────────────────┘     └──────────────────────┘
```

1. **Kurumsal Çerçeve ve Olgunluk Analizi:**  
   $\rightarrow$ [[01 - Kurumsal Strateji, Olgunluk ve ROI]]  
   *IAM Kurumsal Olgunluk Modeli, NIST SP 800-207 Zero Trust uyumu, Kademeli 3 Aşama İlerleme Modeli, Gelecek Yol Haritası (Dinamik Gruplar, IAG) ve Operasyonel Katma Değer Metrikleri.*

2. **Çalıştay ve Paydaş Keşif Kılavuzu:**  
   $\rightarrow$ [[02 - Teknik Keşif Anketi (Workshop Guide)]]  
   *İK (SF), IT Altyapı (Entra ID), SAP Basis (S/4HANA, SCC, SAC) ve Güvenlik ekiplerine yönelik 27 maddelik derinlemesine teknik keşif envanteri (Brownfield SU01 ve Graph API Throttling dahil).*

3. **Çekirdek Sistem Mimarisi ve İş Mantığı:**  
   $\rightarrow$ [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]  
   *BTP Yetkilendirme Stratejisi (JIT Federation vs. IPS), SCIM 2.0 Şema ve Dönüşüm Mantığı Standartları, JML Uç Senaryoları, Yetki Tasarımı vs. Dağıtımı RACI Modeli, S/4HANA Türetilmiş Roller ve BTP CAP ABAC.*

4. **Uygulama ve Entegrasyon Kod Deposu:**  
   $\rightarrow$ [[04 - SAP IPS JSON Kod Spesifikasyonu]]  
   *Microsoft Entra ID kaynak, SAP IAS Identity Directory, SAP S/4HANA ve SAP BTP Subaccount hedef sistemleri için doğrulanmış SCIM 2.0 JSON dönüşüm blokları.*

5. **Güvenlik, Denetim ve Felaket Protokolü:**  
   $\rightarrow$ [[05 - Güvenlik, Denetim ve Break-Glass]]  
   *4 Koruyucu Parametre (Mevcut yetkilerin korunması `authoritative=false`, Toplu silinme eşiği `delete.threshold=10`, Firefighter Filtresi `Z_FF_*`, Değişmez Anahtar `employeeID`), Kasa (Break-Glass), Simülasyon modu ve KVKK/GDPR Uyum Çerçevesi.*

6. **Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi:**  
   $\rightarrow$ [[06 - Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi]]  
   *5 Fazlı Uygulama Yol Haritası: Faz 1 Ağ Tüneli ve Ön Hazırlık (SCC, Brownfield SU01 Temizliği, Entra App Reg, SICF, STRUST), Faz 2 Kimlik Federasyonu ve Güven (OIDC/SAML Trust), Faz 3 IPS Motoru, Faz 4 Runtime Yetki Eşleme (Role Collections, CERTRULE), Faz 5 Test ve Doğrulama (Simülasyon, Drift Yönetimi).*

7. **Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi:**  
   $\rightarrow$ [[07 - Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi]]  
   *6 Senaryolu Saha Yapılabilirlik Matrisi (Day-1, Leaver, Mover, Şirket Kodu Kısıtı, Dış Kaynak, Vekalet), 3 Ekipten Asgari İstenecekler (İK, IT, Basis), Sahadaki 3 Kritik Tıkanma Noktası, Sınır Çizme Kılavuzu (%80 Otomasyon vs %20 Manuel) ve 6 Kritik Day-1/Day-2 Mikro Teknik Risk Analizi.*

8. **Kapsamlı Terimler ve Kısaltmalar Sözlüğü:**  
   $\rightarrow$ [[08 - Terimler ve Kısaltmalar Sözlüğü (Glossary)]]  
   *38 Kritik Kısaltma (IAM, SCIM, OIDC, SAML, ABAC, SoD, FUE, CVI vb.) ve 20 Temel Kavramsal Terim (Birthright, JIT, Principal Propagation, Ephemeral Certificates, Continuous Drift, Shadow User vb.) kurumsal referans kılavuzu.*

9. **Rol İsimlendirme, Yaşam Döngüsü ve İleri Seviye IAM:**  
   $\rightarrow$ [[09 - Rol Standartları, Yaşam Döngüsü ve İleri Seviye IAM Araştırmaları]] • [[sunum_rol_standartlari.html|İnteraktif Sunum]]  
   *PFCG 30 karakter isimlendirme sözdizimi, Composite rol stratejisi, BTP Connectivity ve Principal Propagation (X.509/CERTRULE), BTP IAM (Platform vs İş Kullanıcısı, Role Collections, XSUAA), SAP AI Core ve Joule IAM yönetişimi, SU01 ve IPS varsayılan atamaları, mock test yükleri, CDHDR/CDPOS ve Sentinel SIEM izleme, rol ve unvan emeklilik (deprecation) yaşam döngüsü.*

---

## Kritik Regülasyon ve Uyumluluk Takvimi

> [!IMPORTANT]
> **SAP SuccessFactors Temel Şifreli Girişlerin Kaldırılması (Deprecation of Basic Authentication):**  
> SAP resmi direktifi ve **SAP KBA 2791410** uyarınca, **13 Kasım 2026** tarihi itibarıyla SuccessFactors için temel şifre (Basic Authentication) desteği kalıcı olarak sonlandırılacaktır. Bu tarihten sonra tüm SF kullanıcıları ve API entegrasyonları için **SAP Cloud Identity Services - Identity Authentication (IAS)** kullanımı zorunludur.  
> *Kaynak: [SAP KBA 2791410 - Deprecation of Basic Authentication in SAP SuccessFactors](https://me.sap.com/notes/2791410)*

---

## Konsolide Resmi Standartlar ve Referanslar

| Standart / Kaynak | Resmi Kod / URL | Açıklama / Kapsam |
|:---|:---|:---|
| **SAP KBA 2791410** | [me.sap.com/notes/2791410](https://me.sap.com/notes/2791410) | SuccessFactors Basic Authentication sonlandırılması ve IAS zorunluluğu. |
| **SAP Note 3048590** | [me.sap.com/notes/3048590](https://me.sap.com/notes/3048590) | SuccessFactors ile Identity Authentication entegrasyonu rehberi. |
| **SAP Identity Provisioning Help** | [help.sap.com/docs/identity-provisioning](https://help.sap.com/docs/identity-provisioning) | IPS Transformations, Properties (`ips.delete.threshold`) ve SCIM API motoru. |
| **SAP Identity Directory API** | [api.sap.com/api/IdentityDirectory](https://api.sap.com/api/IdentityDirectory/overview) | SAP resmi SCIM 2.0 şeması: `urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid`. |
| **SAP BTP Security Guide** | [help.sap.com/docs/btp/security](https://help.sap.com/docs/btp/sap-business-technology-platform/security) | XSUAA Role Collection Mapping, OIDC Trust Configuration, Ephemeral Principal Propagation. |
| **SAP CAP Authorization** | [cap.cloud.sap/docs/guides/security/authorization](https://cap.cloud.sap/docs/guides/security/authorization) | CDS seviyesinde deklaratif ABAC (`@restrict: where companyCode = $user.companyCode`). |
| **SAP Cloud Connector Guide** | [help.sap.com/docs/connectivity](https://help.sap.com/docs/connectivity/sap-btp-connectivity-cf/principal-propagation) | X.509 Short-Lived Certificate ile On-Premise ABAP Principal Propagation. |
| **NIST SP 800-207** | [doi.org/10.6028/NIST.SP.800-207](https://doi.org/10.6028/NIST.SP.800-207) | Zero Trust Architecture (PDP/PEP mimarisi, Explicit Verification). |
| **IETF SCIM 2.0 (RFC 7643 / 7644)** | [datatracker.ietf.org/doc/html/rfc7643](https://datatracker.ietf.org/doc/html/rfc7643) | System for Cross-domain Identity Management: Core Schema & Protocol. |
| **Microsoft Entra ID SCIM** | [learn.microsoft.com/entra](https://learn.microsoft.com/en-us/entra/identity/app-provisioning/use-scim-to-provision-users-and-groups) | Graph API SCIM entegrasyonu ve Dinamik Kullanıcı Grupları. |
