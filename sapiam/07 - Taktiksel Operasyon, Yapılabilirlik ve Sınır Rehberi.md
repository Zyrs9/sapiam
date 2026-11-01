---
title: Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi
tags:
  - tactical
  - feasibility
  - operations
  - raci
  - boundaries
  - gotchas
status: approved
version: "3.2"
updated: 2026-10-22
parent: "[[00 - MOC - SAP IAM & JML Strateji Dizini]]"
---

# Taktiksel Operasyon, Yapılabilirlik ve Sınır Rehberi

Taktiksel katman, sistemin teorik vizyonuyla kod seviyesindeki JSON yapılandırması arasında kalan operasyonel köprüdür: **"Hangi senaryo gerçekten çalışır, bunu çalıştırmak için hangi somut şartlar gerekir ve nerede sınır çizeceğiz?"**

İlişkili Dokümanlar:
* Mimari Harita: [[00 - MOC - SAP IAM & JML Strateji Dizini]]
* Keşif Anketi: [[02 - Teknik Keşif Anketi (Workshop Guide)]]
* Çözüm Mimarisi: [[03 - Çekirdek Mimari ve JML Yaşam Döngüsü]]
* JSON Kod Spesifikasyonu: [[04 - SAP IPS JSON Kod Spesifikasyonu]]
* Güvenlik ve Koruyucu Parametreler: [[05 - Güvenlik, Denetim ve Break-Glass]]
* Kurulum Rehberi: [[06 - Uçtan Uca Kurulum, Entegrasyon ve Doğrulama Rehberi]]

---

## 1. Taktiksel Senaryo ve Yapılabilirlik Matrisi

| Senaryo | Yapılabilirlik Derecesi | Sahada Nasıl İşler? | Gerektirdiği Asgari Şartlar | Sınırı / Dikkat Edilecek Nokta |
|:---|:---|:---|:---|:---|
| **1. Standart İşe Giriş (Day-1 Birthright)** | **Tam (%100)** | İK SF'te işe alımı onaylar $\rightarrow$ IT AD'de hesabı ve `SG_SAP_*` grubunu açar $\rightarrow$ IPS kullanıcıyı S/4HANA'da açıp rolü bağlar; BTP'de oturum anında rol koleksiyonu eşleşir. | • SF $\leftrightarrow$ AD entegrasyonunun çalışması.<br>• Hedef sistemde standart PFCG veya BTP rolünün önceden tanımlı olması. | Kullanıcı S/4HANA'da hemen açılsa bile Fiori tarafında işlem yapabilmesi için arka plandaki Business Partner (BP/CVI) senkronizasyonunun tamamlanması gerekir. |
| **2. İşten Ayrılış (Leaver - Erişim Kesme)** | **Tam (%100)** | AD hesabı devre dışı kaldığında (`accountEnabled = false`), IPS hedef sistemlerdeki kullanıcıları kilitler (`active = false`). | • AD'deki durum bilgisinin IPS'e doğru map edilmesi (`$.accountEnabled` $\rightarrow$ `active`). | Standart IPS periyodu (örneğin 30-60 dk) aralığında çalışır. "Acil çıkış" gerekiyorsa ilk 15 dakika için S/4HANA `SU01` ve BTP üzerinden manuel kilitleme bypass akışı tanımlanmalıdır. |
| **3. Departman / Görev Değişimi (Mover)** | **Yüksek (%90)** | SF'te pozisyon değişir $\rightarrow$ AD grubu değişir $\rightarrow$ IPS eski yetkileri geri alır, yeni departmanın rollerini tanımlar. | • AD grubundan çıkarma işleminin zamanında yapılması.<br>• IPS'te ilgili hedef sistemin yetki güncelleme modunun açık olması. | Eski işlerin devri için 10-15 günlük bir yetki çakışması (çift yetki toleransı) gerekiyorsa, AD tarafında eski grubun silinmesi bu süre kadar geciktirilmelidir (Grace Period). |
| **4. Organizasyonel Veri Kısıtı (Şirket Kodu / Masraf Merkezi)** | **Koşullu (%60)** | Kullanıcının sadece kendi şirket kodunu (Örn: `1000`) görmesi ihtiyacı. | • **S/4HANA için:** Türetilmiş Roller (Derived Roles) şarttır.<br>• **BTP için:** AD/IAS profilindeki şirket kodu attribute'unun JWT token'a basılması gerekir. | IPS doğrudan PFCG içindeki organizasyon seviyelerini dolduramaz. S/4HANA yetki ekibi `Z_FI_ACCOUNTANT_1000` gibi türetilmiş rolleri hazır etmezse bu senaryo işletilemez. |
| **5. Dış Kaynak / Danışman / Stajyer Yönetimi** | **Yüksek (%85)** | SF'te kaydı olmayan dış kullanıcıların SAP erişimlerinin yönetilmesi. | • IT'nin bu kişilere AD tarafında son kullanma tarihli (Account Expiration Date) hesap açması.<br>• İlgili SAP güvenlik grubuna eklenmesi. | İK süreci devrede olmadığı için bu hesapların yaşam döngüsü sorumluluğu doğrudan IT Altyapı ve ilgili sözleşme yöneticisinde kalır. |
| **6. Geçici Görev / Acil Durum Erişimi (Vekalet)** | **Hibrit (%50)** | Bir personelin 1 haftalığına başka bir birimin onay ekranına bakması. | • AD tarafında süreli grup üyeliği (veya PIM) **VEYA**<br>• S/4HANA tarafında manuel atama + Firefighter filtresi. | IPS'te `authoritative = true` ise elle verilen geçici roller silinir. Bu rollerin başına `Z_EXC_*` veya `Z_FF_*` konularak IPS kapsamı dışına çıkarılması gerekir. |

---

## 2. Taktiksel Ön Koşullar: Hangi Ekipten Asgari Ne İstenecek?

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

### A. İK / SuccessFactors Ekibinden İstenecekler

1. **Değişmez Personel Numarası (`personIdExternal`):** TC Kimlik veya isim değil; hiçbir koşulda değişmeyen tekil anahtarın ana kaynak olarak teyit edilmesi.
2. **Standart Pozisyon Tanımları:** Unvanların serbest metin yerine sistemdeki standart pozisyon kodlarıyla (`JobCode` / `Position ID`) ilişkilendirilmiş olması.
3. **Mover / Leaver Olay Kodları:** Hangi statü değişiminin (Örn: `TER`, `REH`, `XFR`) sistemde kullanıcı kilitlenmesine veya grup değişimine yol açacağının listesi.

### B. IT / Dizin Hizmetleri (Active Directory / Entra ID) Ekibinden İstenecekler

1. **Grup İsimlendirme Standardı:** SAP gruplarının diğer kurumsal gruplarla karışmaması için kesin bir regex standardı (Örn: `SG_SAP_<SISTEM>_<ROL>`).
2. **Öznitelik (Attribute) Eşleme:** SF'ten gelen personel numarasının AD üzerinde `employeeID` alanına yazıldığının garantilenmesi.
3. **IPS Servis Kaydı (App Registration):** IPS'in Entra ID Graph API'den sadece bu grupları okuyabilmesi için `User.Read.All` ve `GroupMember.Read.All` yetkilerine sahip bir Client ID ve Secret/Sertifika üretilmesi.
4. **Graph API Throttling Koruması ve Filtreleme:** Kurumsal dizindeki binlerce grubun filtresiz çekilerek Microsoft Graph API rate-limit/throttling sınırlarına takılmaması için `entra.id.filter.groups: displayName startswith 'SG_SAP_'` sunucu taraflı filtresinin ve `ips.job.bulk.page.size = 100` sınırının konfigüre edilmesi.

### C. SAP Basis ve Yetki Danışmanlarından İstenecekler

1. **Standart Teknik Rol Envanteri:** Modül danışmanlarının S/4HANA tarafında tasarladığı rollerin kesin teknik isimleri (Örn: `Z_BR_PURCHASER_1010`).
2. **Cloud Connector ve SCIM Tüneli:** BTP ile S/4HANA arasındaki Cloud Connector üzerinde `/sap/bc/scim` yolunun ve servisinin (`SICF`) aktif edilmesi.
3. **İstisna Öneki (Exclusion Prefix):** Manuel veya acil durumlarda verilecek roller için bir ön ek üzerinde mutabakat sağlanması (Örn: `Z_FF_*`, `Z_EXC_*`).
4. **Mevcut (Brownfield) SU01 Kullanıcı Temizliği:** Canlıya geçiş öncesinde mevcut aktif kullanıcıların `SU01` Alias / `externalId` alanlarına İK personel numarasının (`employeeID`) toplu yazılarak SCIM çakışmalarının önlenmesi.

---

## 3. Sahadaki Operasyonel Tıkanma Noktaları ve Pratik Çözümleri

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

## 4. Taktiksel Sınır Çizme Kılavuzu (Neyi Otomatikleştirelim, Neyi Bırakalım?)

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

## 5. Canlıya Geçiş (Day-1) ve Operasyon (Day-2) Mikro Teknik Risk Analizi

Canlıya geçiş anında ve takip eden operasyonel süreçte sistemleri kilitleyebilecek 6 kritik mikro teknik risk ve sahada uygulanması gereken önlemler aşağıda analiz edilmiştir:

| Risk Alanı | Teknik Kök Neden ve Semptom | Operasyonel Etki (Day-1 / Day-2) | Zorunlu Teknik Önlem / Çözüm |
|:---|:---|:---|:---|
| **1. Brownfield (Mevcut) SU01 Kullanıcılarının Eşleşmemesi** | Mevcut S/4HANA sisteminde önceden elle açılmış kullanıcıların `SU01` Alias / `externalId` alanlarında personel numarasının (`personIdExternal`) bulunmaması. | IPS senkronizasyonu çalıştığında kullanıcıyı `externalId` üzerinden bulamaz; aynı `userName` ile yeni kayıt açmaya çalışırken çakışma yaşanır ve provizyon hata verir. | Canlıya geçiş öncesinde SAP Basis ekibi mevcut aktif `SU01` kullanıcılarının `externalId` / `Alias` alanlarını İK personel numarasıyla (`employeeID`) eşitlemelidir. |
| **2. S/4HANA CVI ve Business Partner (BP) Senkronizasyon Gecikmesi** | S/4HANA mimarisinde her çalışanın aynı zamanda bir İş Ortağı (Business Partner - BP) olması zorunludur (SAP Note 2570961). | IPS, `SU01` hesabını ve PFCG rolünü saniyeler içinde açar; ancak arka plandaki HCM $\leftrightarrow$ BP (`/BOM/`) senkronizasyon işi çalışmadıysa kullanıcı Fiori ekranlarında "İş Ortağı Bulunamadı" hatası alır. | S/4HANA üzerindeki `/BOM/` (CVI/HCM Employee Sync) arka plan işi, IPS çalışma periyodundan daha sık veya hemen ardışık çalışacak şekilde zamanlanmalıdır. |
| **3. Entra ID Graph API Rate-Limit / Throttling ve Nested Groups** | Kurumsal dizindeki binlerce grubun filtresiz çekilmesi veya iç içe grupların (`expandNestedGroups`) derinlemesine çözümlenmesi. | IPS senkronizasyon süresi uzar, ağ trafiği şişer ve Microsoft Graph API üzerinde paket limitlerine takılarak senkronizasyon yarıda kesilir. | Kaynak sistem özelliklerinde `entra.id.filter.groups: displayName startswith 'SG_SAP_'` sunucu taraflı filtresi ve `ips.job.bulk.page.size = 100` sınırı zorunlu tutulmalıdır. |
| **4. S/4HANA Sözleşmesel Lisans (FUE) Tipi Aşımı** | IPS üzerinden açılan kullanıcılara `SU01` üzerinde varsayılan olarak yüksek maliyetli kullanıcı tipi (*Professional*) atanması. | Sadece izin talebi (ESS) veya temel rapor kullanacak personelin tam lisans havuzunu tüketmesi sonucu sözleşmesel FUE (Full User Equivalent) maliyeti aşılır. | S/4HANA tarafında varsayılan sözleşmesel kullanıcı tipi (*Professional*, *Functional*, *Productivity*) modül yetki ekibiyle önceden mutabık kalınarak sınırlandırılmalıdır. |
| **5. Koruyucu Modda Yetki Kayması (Continuous Drift)** | Geçiş fazında mevcut manuel rolleri korumak için `s4hana.authoritative.role.management = false` parametresinin kullanılması. | `authoritative = false` iken `SU01` üzerinden el altından verilen roller IPS tarafından silinmez; zamanla Entra ID grupları ile S/4HANA gerçek yetkileri birbirinden kopar. | Sistem kararlı hale geldikten sonra ayda bir kez IPS dışından elle atanmış yetkilerin denetim raporu alınmalı ve temizlik sonrası kademeli olarak `true` moduna geçilmelidir. |
| **6. Zamanlanmış Senkronizasyon Gecikmesi ve Acil Çıkışlar** | IPS zamanlanmış (scheduled) bir SCIM ETL motoru olarak belirli periyotlarla (örn. 30-60 dk) çalışır. | Güvenlik riski taşıyan acil işten çıkışlarda (Immediate Revocation) bir sonraki IPS döngüsünü beklemek 15 dakikalık SLA hedefini ihlal edebilir. | Acil çıkış durumlarında ilk 15 dakika içinde oturumu sonlandırmak için S/4HANA `SU01` ve BTP üzerinden manuel kilitleme bypass prosedürü tanımlanmalıdır. |
