---
title: SAP IPS JSON Kod Spesifikasyonu
tags:
  - ips
  - scim
  - json
  - transformation
  - integration
  - s4hana
  - btp
status: approved
version: "3.0"
updated: 2026-10-19
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# SAP IPS JSON Kod Spesifikasyonu

Bu belge, SAP Cloud Identity Services - Identity Provisioning (IPS) motorunda çalışan, **IETF RFC 7643/7644 SCIM 2.0** ve **SAP Identity Directory** standartlarına tam uyumlu JSON dönüşüm kodlarını içerir.

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* Güvenlik ve Denetim: [[05 - Güvenlik, Denetim ve Break-Glass]]

---

## 1. Kaynak Sistem (Source System): Microsoft Entra ID

Bu dönüşüm bloğu:
1. Artımlı okuma (Delta Sync) için gerekli `entityIdTargetSystem` parametrelerini taşır [^1].
2. Sadece `SG_SAP_` öneki ile başlayan kurumsal güvenlik gruplarını IPS kapsamına alır [^2].
3. SuccessFactors kökenli `employeeId` bilgisini tekil correlation ID olarak eşler.

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

## 2. Hedef Sistem (Target): SAP Cloud Identity Services - IAS (Identity Directory)

Bu dönüşüm bloğu, kullanıcıları ve grupları merkezi SAP Identity Directory deposuna resmi SAP SCIM 2.0 uzantı şemasıyla (`urn:ietf:params:scim:schemas:extension:sap:2.0:User:userUuid`) kaydeder [^3].

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

## 3. Hedef Sistem (Target): SAP S/4HANA (SCIM 2.0 / Cloud Connector)

Bu dönüşüm bloğu, standart **`$.roles[+].value`** (append) sözdizimi ile S/4HANA içerisindeki PFCG Türetilmiş Rollerini (Derived Roles) kullanıcıya atar [^4].

> [!WARNING]
> SCIM 2.0 standardında yeni bir dizi elemanı eklenirken hedef yol filtreleme (`[?(@.value == '...')]`) kullanılamaz; dizinin sonuna ekleme yapan `[+]` operatörü zorunludur.

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

## 4. Hedef Sistem (Target): SAP BTP Subaccount (XSUAA / Role Collection Target)

Bu dönüşüm bloğu, BTP Subaccount seviyesinde ön-provizyon gerektiren yönetici veya servis hesaplarına **Role Collection** ataması yapar [^5].

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

## Resmi Referanslar ve Kaynakça

[^1]: **SAP Help Portal - Identity Provisioning:** *Manage Delta Reading of Entities*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/manage-delta-reading-of-entities](https://help.sap.com/docs/identity-provisioning/identity-provisioning/manage-delta-reading-of-entities)
[^2]: **SAP Help Portal - Identity Provisioning:** *Transformation Operations and Expressions*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/transformation-expressions](https://help.sap.com/docs/identity-provisioning/identity-provisioning/transformation-expressions)
[^3]: **SAP Business Accelerator Hub:** *Identity Directory SCIM 2.0 REST API Specification*. [https://api.sap.com/api/IdentityDirectory/overview](https://api.sap.com/api/IdentityDirectory/overview)
[^4]: **IETF RFC 7644:** *System for Cross-domain Identity Management: Protocol (Section 3.5.2 Modifying with PATCH/Append)*. [https://datatracker.ietf.org/doc/html/rfc7644#section-3.5.2](https://datatracker.ietf.org/doc/html/rfc7644#section-3.5.2)
[^5]: **SAP Help Portal - Identity Provisioning:** *SAP BTP Subaccount Target System Configuration*. [https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-btp-subaccount](https://help.sap.com/docs/identity-provisioning/identity-provisioning/sap-btp-subaccount)
