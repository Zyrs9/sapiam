# SAP Kurumsal Sistem Mimarisi ve Bilgisayar Bilimleri El Kitabı
**Sürüm:** 2.1 (Master Architecture Document - Consolidated)  
**Kapsam:** Bare-Metal Donanımdan Dağıtık Bulut Bilişime, SAP HANA Çekirdeğinden Kurumsal Yapay Zekaya (Full Stack)

---

## 1. Donanım, Sanallaştırma ve İşletim Sistemi Altyapısı (Infrastructure & OS)

SAP sistemlerinin omurgasını oluşturan **SAP HANA**, disk tabanlı geleneksel RDBMS mimarilerinin getirdiği I/O darboğazlarını aşmak amacıyla doğrudan modern CPU mimarisi, bellek adresleme mantığı ve işletim sistemi çekirdeğine entegre olacak şekilde tasarlanmıştır.

```
+---------------------------------------------------------------------------------+
|                                 SAP HANA Runtimes                               |
+---------------------------------------------------------------------------------+
| POSIX Shared Memory (/dev/shm) | NUMA-Aware Threads | Direct Memory Pointers    |
+---------------------------------------------------------------------------------+
| Linux Kernel: SLES for SAP / RHEL for SAP (hugepages, sysctl, transparent HP)   |
+---------------------------------------------------------------------------------+
| Hypervisor (VMware ESXi / KVM / PowerVM) VEYA Bare-Metal (x86-64 / IBM Power)   |
+---------------------------------------------------------------------------------+
| Hardware: NUMA Node 0 (CPUs + RAM) <--- QPI/UPI ---> NUMA Node 1 (CPUs + RAM)   |
+---------------------------------------------------------------------------------+
```

### 1.1. İşletim Sistemi ve Çekirdek (Kernel) Özelleştirmeleri

*   **Desteklenen Dağıtımlar:** SAP HANA, sıradan genel amaçlı dağıtımlarda çalışmaz; yalnızca çekirdeği SAP iş yükleri için önceden optimize edilmiş **SUSE Linux Enterprise Server for SAP Applications (SLES for SAP)** veya **Red Hat Enterprise Linux for SAP Solutions (RHEL for SAP)** üzerinde desteklenir.
*   **Bellek Yönetimi ve NUMA (Non-Uniform Memory Access):**
    *   Terabaytlarca RAM içeren çok soketli modern kurumsal sunucularda her CPU soketinin doğrudan bağlı olduğu yerel bir bellek havuzu (NUMA node) bulunur. Bir CPU çekirdeğinin uzak soketteki belleğe erişmesi veri yolu (Intel UPI / AMD Infinity Fabric) gecikmesi yaratır.
    *   HANA'nın bellek yöneticisi (**NUMA-aware memory allocator**), veritabanı iş parçacıklarını (threads) verinin bulunduğu yerel NUMA düğümüne kilitler (thread affinity ve CPU pinning).
*   **Transparent Huge Pages (THP) Paradoksu:**
    *   Linux çekirdeğindeki standart THP mekanizması, sanal bellek sayfalarını dinamik olarak $2\text{ MB}$ veya $1\text{ GB}$'lık bloklara birleştirmeye çalışır. Ancak yoğun yazma ve serbest bırakma döngülerinde arka planda çalışan `khugepaged` servisi yüksek bellek parçalanmasına (fragmentation) ve çekirdek kilitlenmelerine (lock contention / latency spikes) yol açar.
    *   Bu nedenle SAP HANA çalışan tüm sistemlerde THP çekirdek seviyesinde (`transparent_hugepage=never`) kesinlikle kapatılır. Sayfa tablosu şişmesini engellemek için doğrudan statik HugePage yapılandırmaları tercih edilir.
*   **Linux Parametreleri ve Dosya Sistemleri:**
    *   `vm.max_map_count`: HANA terabaytlarca bellek segmentini doğrudan adreslediğinden, işletim sisteminin izin verdiği maksimum bellek haritalama sayısı tipik sistemlerde `2147483648` (2 milyar) seviyesine çekilir.
    *   `XFS`: Veri (`/hana/data`) ve günlük (`/hana/log`) dosya sistemlerinde doğrudan asenkron I/O (Direct I/O / `O_DIRECT`) desteği sunan `XFS` standart dosya formatıdır.
    *   `/dev/shm`: Süreçler arası yüksek hızlı veri aktarımı için POSIX Shared Memory kullanılır.

### 1.2. Donanım Tedarik Modelleri: Appliance vs. TDI

SAP HANA altyapısı iki katı sertifikasyon modeline tabidir:

*   **Appliance Modeli:** Donanım üreticisi (Dell, HPE, Lenovo) tarafından CPU, RAM, disk ve ağ bileşenleri önceden tanımlanmış; işletim sistemi ve HANA imajı fabrikada kurulmuş kapalı kutudur. Esnekliği sıfırdır; harici depolama ünitelerine bağlanamaz.
*   **TDI (Tailored Datacenter Integration):** Kurumsal şirketlerin kendi veri merkezlerindeki sertifikalı depolama ağlarını (SAN/NAS - Pure Storage, NetApp, Dell EMC, HPE Primera) ve ağ anahtarlarını kullanmasına izin veren modeldir.
*   **KPI Doğrulama Araçları (`HWCCT` / `HCMT`):** Bir donanımın TDI kapsamında canlıya alınabilmesi için SAP'nin **HCMT (HANA Cloud Migration Tool / Hardware Configuration Check Tool)** testini geçmesi zorunludur.
    *   *Kritik SAP Eşiği:* Log diski (`/hana/log`) üzerinde $4\text{ KB}$ ve $1\text{ MB}$ blok boyutlarında ardışık yazma gecikmesi kesinlikle **$1\text{ ms}$'nin altında** olmak zorundadır. Aksi halde ACID commit işlemleri gecikir ve kurulum sertifikasyonu reddedilir.

### 1.3. Ölçekleme Modelleri: Scale-Up vs. Scale-Out Sınırları

```
        SCALE-UP (Dikey Ölçekleme)                      SCALE-OUT (Yatay Ölçekleme)
   +------------------------------------+      +-----------------------------------------+
   |        Tek Bir Büyük Node          |      |  Node 1 (Master)      Node 2 (Worker)   |
   |                                    |      |  [SystemDB/Catalog]   [Part. ACDOCA]    |
   | - 4 / 8 / 16 CPU Soketi            |      |         |                     |         |
   | - 1.5 TB - 24 TB RAM               |      |         +----------+----------+         |
   | - Sıfır Ağ İletişim Gecikmesi      |      |                    |                    |
   |                                    |      |   Düğümler Arası 100GbE Interconnect    |
   | * S/4HANA (OLTP/HTAP) ZORUNLULUĞU  |      |                    |                    |
   |                                    |      |  Node 3 (Worker)      Node 4 (Standby)  |
   +------------------------------------+      |  [Part. ACDOCA]       [Failover Hazır]  |
                                               |                                         |
                                               | * SAP BW/4HANA (OLAP) MİMARİSİ          |
                                               +-----------------------------------------+
```

*   **Scale-Up (S/4HANA Standardı):** OLTP sistemlerinde sürekli tekil satır kilitleri (row-level locks) ve anlık tablo birleştirmeleri (join) yapılır. Ağ üzerinden iki farklı sunucudaki bellek segmentini birleştirmek gecikmeyi milisaniyeler seviyesine çıkaracağından; SAP, tüm **S/4HANA üretim ortamlarının tek bir devasa sunucuda (Scale-Up)** çalışmasını şart koşar.
*   **Scale-Out (BW/4HANA & Analitik Sistemler):** Analitik veri ambarlarında veriler devasa boyutlara ulaştığında (onlarca TB), veriler düğümlere bölünür (partitioning). Master düğüm metaveriyi tutarken, Worker düğümleri veriyi paralel olarak işler. Standby düğüm ise donanım arızasında devreye girmek üzere hazır bekler.

### 1.4. Yüksek Erişilebilirlik (HA/DR): HSR, Pacemaker ve STONITH

Veritabanı seviyesindeki iş sürekliliği **HANA System Replication (HSR)** ve Linux kümeleme yazılımlarıyla sağlanır:

```
+-----------------------------------+                   +-----------------------------------+
|     BİRİNCİL DÜĞÜM (Primary)      |                   |     İKİNCİL DÜĞÜM (Secondary)     |
|                                   |                   |                                   |
| [Transaction] ---> [Redo Log Buf] |                   | [Log Receiver]                    |
|                          |        |                   |       |                           |
|                          v        |   HSR Bağlantısı  |       v                           |
|                    /hana/log      | =================>|  /hana/log (Replay Engine)        |
+-----------------------------------+                   +-----------------------------------+
                  \                                       /
                   \                                     /
            +---------------------------------------------------+
            |    OS Kümeleme Katmanı (Pacemaker / Corosync)     |
            |    - HANA Resource Agents (SAPHanaSR)             |
            |    - Fencing Mekanizması: STONITH (IPMI / iLO)    |
            +---------------------------------------------------+
```

#### HSR Replikasyon Modları
1.  **`sync` (Senkron):** Birincil düğümdeki işlem, redo log paketi ikincil düğüme ulaşıp ikincil düğümün NVMe/SSD diskine kalıcı olarak yazılana kadar commit edilmez. Veri kaybı sıfırdır ($RPO = 0$).
2.  **`syncmem` (Belleğe Senkron):** Redo log ikincil düğümün RAM'ine ulaştığı anda birincil düğüme onay döner; diske yazma beklenmez. Ağ bant genişliği yüksekse daha düşük işlem gecikmesi sunar ($RPO = 0$).
3.  **`async` (Asenkron):** Birincil düğüm ikincil düğümün yanıtını beklemeden işlemi commit eder. Düğümler arasındaki mesafe $100\text{ km}$'yi aştığında (Felaket Kurtarma Merkezi - FKM senaryolarında) hat gecikmesinin sistemi yavaşlatmaması için tercih edilir ($RPO > 0$).
4.  **`full sync`:** İkincil düğüm erişilemez hale geldiğinde birincil düğümdeki tüm veritabanı yazma operasyonlarını dondurur. Veri tutarsızlığını ve çatallanmayı (split-brain) kesin olarak engeller.
5.  **Active/Active Read-Enabled (`logreplay_readaccess`):** İkincil düğümün atıl beklemesini önler; ikincil düğüm portuna salt-okunur (read-only) sorgular gönderilerek raporlama iş yükleri birincil sistemden izole edilir.

#### Kümeleme ve STONITH Mimarisi
*   SLES veya RHEL üzerinde koşan **Pacemaker + Corosync**, `SAPHanaSR` ajanları vasıtasıyla birincil düğümün durumunu saniyede bir kontrol eder.
*   Birincil düğüm çöktüğünde sanal IP (Virtual IP) anında ikincil düğüme yönlendirilir.
*   **STONITH (Shoot The Other Node In The Head):** Her iki düğümün de aynı anda aktif olup aynı depolama alanına yazarak veriyi bozmasını (split-brain) engellemek için; hayatta kalan düğüm IPMI, iLO veya vCenter API aracılığıyla arızalı düğümün elektriğini donanım seviyesinde anında keser.

### 1.5. İşlemci Mimarisi Karşılaştırması: x86-64 vs. IBM Power (ppc64le)

| Donanım Parametresi | Intel Xeon / AMD EPYC (x86-64) | IBM Power9 / Power10 (ppc64le) |
| :--- | :--- | :--- |
| **İş Parçacığı / Çekirdek (SMT)** | Hyper-Threading (SMT2: 1 çekirdek = 2 thread) | **SMT8** (1 fiziksel çekirdek = 8 eşzamanlı donanım thread'i) |
| **Bellek Bant Genişliği** | $\sim 200 - 300\text{ GB/s}$ (DDR4/DDR5) | **$800 - 1000\text{ GB/s}+$** (Open Memory Interface - OMI) |
| **Sanallaştırma Katmanı** | VMware ESXi / KVM (Hipervizör yükü $\sim 2-4\%$) | **IBM PowerVM** (Donanım seviyesinde mikro-bölümleme, sıfır OS yükü) |
| **Dinamik Kaynak Yönetimi** | Sanal makine yeniden başlatma gerektirebilir | Canlı dinamik CPU/RAM ekleme ve çıkarma (Hot-add) |

---

## 2. Veritabanı Çekirdeği: SAP HANA Internals, Bellek ve Veri Katmanlaması

SAP HANA, yalnızca bir SQL sorgulayıcısı değil; hesaplamayı diske veri taşıyarak değil, verinin bellekte durduğu yerde (code pushdown) gerçekleştiren bir paralel hesaplama platformudur. Analiz ve işlem yüklerini aynı bellek bloklarında çalıştıran bir **HTAP (Hybrid Transaction/Analytical Processing)** motorudur.

```
                           +------------------------+
                           |  SQL Client / OData    |
                           +-----------+------------+
                                       |
+--------------------------------------v------------------------------------------+
| SAP HANA Indexserver (Ana Süreç / Daemon)                                       |
|                                                                                 |
|  +--------------------+  +----------------------+  +-------------------------+  |
|  | SQL Parser / Auth  |  |  HEX Engine (LLVM)   |  | Plan Optimizer / Codegen|  |
|  +---------+----------+  +----------+-----------+  +------------+------------+  |
|            |                        |                           |               |
|  +---------v------------------------v---------------------------v------------+  |
|  |               Execution Engine (Vectorized Operations, SIMD)              |  |
|  +------------------------------+-------------------------------+------------+  |
|                                 |                                               |
|             +-------------------+-------------------+                           |
|             v                                       v                           |
|  +-----------------------+              +-----------------------+               |
|  |   Row Store (RAM)     |              |  Column Store (RAM)   |               |
|  | (Config/Meta, OLTP)   |              | (Ana Tablolar, OLAP)  |               |
|  +-----------------------+              +-----------+-----------+               |
|                                                     |                           |
|                                       +-------------+-------------+             |
|                                       v                           v             |
|                             +-------------------+       +-------------------+   |
|                             | Main Memory Area  | <==== | Delta Memory Area |   |
|                             | (Sıkıştırılmış, RO| Merge | (Yazılabilir,     |   |
|                             |  ve Vektörize)    | Oper. |  Satır-tabanlı)   |   |
|                             +-------------------+       +-------------------+   |
+---------------------------------------+---------------------------+-------------+
                                        |                           |
               (Asenkron Savepoint - ~5dk)          (Senkron Redo Log - WAL)
                                        |                           |
                                        v                           v
                       +--------------------+      +--------------------+
                       |    /hana/data      |      |     /hana/log      |
                       | (Disk Persistence) |      | (SSD/NVMe WAL Log) |
                       +--------------------+      +--------------------+
```

```
+----------------------------------------------------------------------------------------------------+
| SAP HANA Indexserver Memory Layout (Linux Virtual Address Space)                                   |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | Physical RAM Size (Host)                                                                     |  |
|  +----------------------------------------------------------------------------------------------+  |
|  | Global Allocation Limit (GAL - HANA Tarafından Ayrılabilir Tavan RAM Sınırı)                 |  |
|  +----------------------------------------------------------------------------------------------+  |
|  | Pool Allocators (Shared/Process Context Memory, Heap, Query Compilation Buffer)             |  |
|  +----------------------------------------------------------------------------------------------+  |
|  | Column Store Tablo Bellek Alanı (Tablolar, Sütunlar, İndeksler)                             |  |
|  |                                                                                              |  |
|  |  +------------------------------------------+  +------------------------------------------+  |  |
|  |  | Main Store (Okuma-Odaklı)                |  | Delta Store (Yazma-Odaklı)               |  |  |
|  |  | - Devasa, Yüksek Sıkıştırmalı Veri       |  | - Küçük, Satır-tabanlı / Hafif Sıkışık   |  |  |
|  |  | - Sıralı Sözlük (Sorted Dictionary)      |  | - Sırasız Sözlük (Unsorted Dict)         |  |  |
|  |  | - Bit-Packed Attribute Vectors           |  | - Write-Optimized Memory Structure       |  |  |
|  |  +------------------------------------------+  +------------------------------------------+  |  |
|  +----------------------------------------------------------------------------------------------+  |
+----------------------------------------------------------------------------------------------------+
```

### 2.1. Sütun Depolama Veri Yapıları (Column Store Internals)

Bir sütun (örneğin `Ulke` sütunu) bellekte ham metin dizisi olarak tutulmaz. Üç ayrı veri yapısına parçalanır:

```
Veri: ["TR", "DE", "TR", "US", "TR"]

1. Sorted Dictionary (Benzersiz Değerler Dizisi)
   ValueID | Value
   --------+------
   0       | "DE"
   1       | "TR"
   2       | "US"

2. Attribute Vector (Bit-Packed Dizi - Gerçek Satırlar)
   RowID | ValueID (Binary)
   ------+-----------------
   0     | 01 (TR)
   1     | 00 (DE)
   2     | 01 (TR)
   3     | 10 (US)
   4     | 01 (TR)
```

*   **Sorted Dictionary:** Benzersiz değerlerin alfabetik veya sayısal sıralı dizisidir. Her benzersiz değere 0'dan başlayan bir tamsayı belirteç (`ValueID`) atanır.
*   **Bit-Packed Attribute Vector:** Orijinal satır sırasını koruyan dizidir; ancak değerin kendisini değil, sadece `ValueID`'sini tutar.
    *   3 benzersiz değer için sadece 2 bit (`00`, `01`, `10`) yeterlidir.
    *   CPU, `SELECT COUNT(*) WHERE Ulke = 'TR'` sorgusu geldiğinde metin karşılaştırması yapmaz; SIMD komutlarıyla doğrudan `01` bit maskesini vektörel olarak tarar.
*   **Sıkıştırma Algoritmaları:** Sütun verileri sözlük kodlaması (Dictionary Encoding) yanı sıra Run-Length Encoding (RLE), Cluster Encoding, Sparse Encoding ve Prefix Encoding ile sıkıştırılır. Sayısal filtrelemeler doğrudan CPU L1/L2 önbelleğinde, SIMD (AVX-512) vektör komutları kullanılarak bellekten veri çözülmeden (**decompression-free execution**) yürütülür.

### 2.2. Delta Store ve Delta Merge Algoritması

Sıkıştırılmış ve sıralı ana tabloya (`Main Store`) doğrudan yeni bir satır eklemek, tüm sözlüğün yeniden sıralanmasını ve attribute vector bitlerinin kaydırılmasını gerektirir; bu işlem CPU ve bellek bant genişliğini kilitler. Bu yüzden yazma işlemleri geçici bir tampon olan **Delta Store**'a yazılır.

```
BAŞLANGIÇ DURUMU:
[ Okuma İstekleri ] ----> Main 1 + Delta 1
[ Yazma İstekleri ] ----> Delta 1

ADIM 1: TOKEN TRANSFERİ VE AYIRMA (SPLIT)
[ Okuma İstekleri ] ----> Main 1 + Delta 1 + Delta 2
[ Yazma İstekleri ] ----> Delta 2 (Yeni açılan yazılabilir alan)
Delta 1 (Salt-okunur olarak dondurulur)

ADIM 2: BİRLEŞTİRME VE YENİ MAIN ÜRETİMİ (MERGE IN MEMORY)
+------------+       +------------+
|   Main 1   |   +   |  Delta 1   |  ========>  [ Yeni Main 2 Üretimi ]
+------------+       +------------+             - Sözlükler birleştirilir ve yeniden sıralanır.
                                                - Yeni bit-packed attribute vector oluşturulur.
                                                - Silinmiş satırlar (MVCC) temizlenir.

ADIM 3: DEĞİŞTİRME VE COMMIT (SWITCH)
[ Okuma İstekleri ] ----> Main 2 + Delta 2
[ Yazma İstekleri ] ----> Delta 2

ADIM 4: ESKİ SEGMENTLERİN SERBEST BIRAKILMASI (CLEANUP)
Main 1 ve Delta 1 bellekten (RAM) boşaltılır (Free memory).
Main 2 asenkron savepoint ile kalıcı diske (/hana/data) yazılır.
```

*   **Tetikleme Kriterleri (Auto Merge Check - `Mergedog`):**
    *   *Delta Boyutu:* Delta belleği Main'in $\%5 - \%10$'unu veya mutlak MB sınırını aştığında.
    *   *Log Alanı Tüketimi:* Delta günlüğe aşırı yazma yapıldığında.
    *   *Sorgu Maliyeti:* Delta aramasının CPU tarama maliyeti eşiği aştığında.
*   **Geçici Bellek Çiftlenmesi Riski (Memory Footprint Spike):**
    *   Adım 2 sırasında, $Main_1$ ve $Delta_1$ hala aktif sorgulara hizmet vermek için RAM'de tutulurken, aynı anda yeni oluşturulan $Main_2$ de belleğe yazılır.
    *   Bu durum, birleştirme anında tablonun bellekte **yaklaşık 2 kat yer kaplamasına** ($Size(Main_1) + Size(Delta_1) + Size(Main_2)$) yol açar.
    *   Eğer bellek tahsisi limitlere yakınsa, işletim sistemi seviyesinde **OOM (Out-of-Memory Killer)** tetiklenebilir.
*   **Bölümleme Çözümü (Table Partitioning):**
    *   Yüz milyonlarca satır içeren dev tablolar (`ACDOCA`, `MATDOC`), HASH veya RANGE yöntemleriyle fiziksel bölümlere (partitions) ayrılır. Delta merge yalnızca ilgili küçük bölüme uygulanarak hem işlem süresi milisaniyelere çekilir hem de bellek zıplamaları kontrol altında tutulur.

### 2.3. Kalıcılık (Persistence) ve Veri Güvenliği (ACID)

HANA'nın bellekte çalışması veri kaybı riskini doğurmaz; iki aşamalı disk kalıcılığı kullanılır:
1.  **Redo Log Buffer (`/hana/log`):** Bir işlem `COMMIT` edildiğinde, sadece yapılan operasyon sıralı olarak diske (Write-Ahead Logging - WAL) yazılır. Bu yazma NVMe disk üzerinde senkron gerçekleşir.
2.  **Savepoints (`/hana/data`):** Her 5 dakikada bir, bellekteki değişmiş sayfalar (dirty pages) toplu ve asenkron biçimde diske aktarılır. Sistem çökerse en son savepoint diske yüklenir ve ardından gelen loglar replay edilerek ACID garantisi korunur.

### 2.4. Multi-Tenant Database Containers (MDC) ve Daemon Filosu

HANA 2.0+ ile birlikte MDC zorunlu hale getirilmiştir:

```
+----------------------------------------------------------------------------------------------------+
| SAP HANA 2.0+ MULTI-TENANT DATABASE CONTAINER (MDC) MİMARİSİ                                      |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | SYSTEM DATABASE (SystemDB - Port: 3<NR>13)                                                    |  |
|  | Süreçler:                                                                                    |  |
|  | - hdbnameserver: Küresel küme topolojisi, kiracı kataloğu, lisanslama, HSR orkestrasyonu        |  |
|  | - hdbcompileserver: SQLScript ve ABAP CDS için dinamik C++ kod derleme motoru                |  |
|  | - hdbpreprocessor: Metin arama, fuzzy search ve doküman indeksleme motoru                    |  |
|  | - hdbwebdispatcher: HTTP/HTTPS REST trafiğini ilgili Tenant Indexserver'a yönlendirir       |  |
|  +----------------------------------------------------------------------------------------------+  |
|                                                 |                                                  |
|           +-------------------------------------+-------------------------------------+            |
|           v                                                                           v            |
|  +---------------------------------------+           +---------------------------------------+     |
|  | TENANT DATABASE 1: S/4HANA (PRD)      |           | TENANT DATABASE 2: ANALYTICS / BTP    |     |
|  | Port: 3<NR>15                         |           | Port: 3<NR>40                         |     |
|  | Süreçler:                             |           | Süreçler:                             |     |
|  | - hdbindexserver (İzole Bellek Alanı) |           | - hdbindexserver (İzole Bellek Alanı) |     |
|  | - hdbdiserver (HDI Container Dağıtımı)|           | - hdbdiserver (HDI Container Dağıtımı)|     |
|  | Veri Alanı: ACDOCA, MATDOC vb.        |           | Veri Alanı: Analitik modeller         |     |
|  +---------------------------------------+           +---------------------------------------+     |
+----------------------------------------------------------------------------------------------------+
```

*   **SystemDB:** İş verisi tutmaz; topolojiyi, CPU/RAM kotalarını, lisanslamayı ve kiracı izolasyonunu yönetir.
*   **Tenant DB:** Uygulama verilerinin koştuğu izole konteynerdir. Kendi bağımsız `hdbindexserver` sürecine, bağlantı portlarına (`3<InstanceNo>15`) ve bellek sınırına sahiptir.
*   **`hdbdiserver` (Deployment Infrastructure Server):** HDI (HANA Deployment Infrastructure) mimarisinde tasarım zamanı (design-time) nesnelerini (CDS, Calculation View, Tablo tanımları) bağımsız şemalarda derleyip çalışma zamanı nesnelerine dönüştürür.
*   **HEX (HANA Execution Engine):** Eski Calculation Engine ve Row Engine yerine geçen yeni nesil motordur. CDS View sorgularını doğrudan LLVM ile JIT (Just-In-Time) derleyerek C++ makine koduna dönüştürür ve CPU vektör yazmaçlarına (AVX-512) doğrudan haritalar.

### 2.5. Veri Sıcaklığı Katmanlaması (Data Tiering): Hot, Warm, Cold

*   **Hot Data (DRAM Column Store):** Son 1-2 yılın aktif verisi (ACDOCA, cari kayıtlar). Tamamen RAM'dedir; mikrosaniye gecikmeyle taranır.
*   **Warm Data (Native Storage Extension - NSE):** 2-5 yıllık daha az erişilen veriler. Tablonun tamamı RAM'e yüklenmez; sayfa tabanlı (`PAGE LOADABLE`) olarak NVMe diskte durur. Sadece sorgunun dokunduğu $4\text{ MB}$'lık sayfalar RAM'deki **Buffer Cache** alanına alınır. DRAM ihtiyacını $\%40 - \%60$ düşürür.
*   **Cold Data (HANA Data Lake / Nesne Depolama):** 5 yıldan eski arşiv verisi. AWS S3 veya Azure Blob Storage üzerinde tutulur. Saniyeler seviyesinde gecikmeyle çalışır; terabayt başına maliyeti düşüktür.

---

## 3. Uygulama ve Çalışma Zamanı: ABAP Platformu ve Dağıtık Topoloji

ABAP Platformu (tarihsel adıyla NetWeaver AS ABAP), yüksek eşzamanlılıklı kurumsal iş yüklerini yönetmek üzere C/C++ ile yazılmış hibrit bir sanal makinedir.

```
                                  [ Dış Trafik / Web Dispatcher ]
                                                 |
                         +-----------------------+-----------------------+
                         |                                               |
+------------------------v--------------------+ +------------------------v--------------------+
| PRIMARY APPLICATION SERVER (PAS)            | | ADDITIONAL APPLICATION SERVER (AAS)         |
| - Dispatcher & Work Process Havuzu          | | - Dispatcher & Work Process Havuzu          |
| - Internet Communication Manager (ICM)      | | - Internet Communication Manager (ICM)      |
| - Yerel Table Buffer (Shared Memory)        | | - Yerel Table Buffer (Shared Memory)        |
+------------------------+--------------------+ +------------------------+--------------------+
                         |                                               |
                         +-----------------------+-----------------------+
                                                 |
             +-----------------------------------v-----------------------------------+
             | ABAP CENTRAL SERVICES (ASCS) - YÜKSEK ERİŞİLEBİLİR KÜMELEME           |
             |                                                                       |
             |  +--------------------------------+  +-----------------------------+  |
             |  | Message Server (msg_server)    |  | Standalone Enqueue 2 (ENSA2)|  |
             |  | - PAS/AAS yük dengeleme (LB)   |  | - Global Kilit Tablosu      |  |
             |  | - Düğümler arası haberleşme    |  |   (In-Memory Lock Table)    |  |
             |  +--------------------------------+  +--------------+--------------+  |
             |                                                     |                 |
             |                       (Bellek Replikasyonu)         v                 |
             |  +--------------------------------------------------------------+  |  |
             |  | Enqueue Replication Server 2 (ERS2)                          |  |  |
             |  | - İkincil düğümde gölge kilit tablosunu RAM'de senkron tutar |  |  |
             |  +--------------------------------------------------------------+  |  |
             +-----------------------------------------------------------------------+
```

### 3.1. Dağıtık Topoloji: ASCS, PAS ve AAS Ayrımı

*   **ASCS (ABAP Central Services):** Sistemin küresel koordinasyon merkezidir:
    *   **Message Server:** Uygulama sunucuları (PAS/AAS) arasındaki durum sinyallerini (heartbeat) dinler; gelen kullanıcı trafiğini en boş sunucuya yönlendirir.
    *   **Standalone Enqueue Server 2 (ENSA2):** Tüm sistemdeki mantıksal iş kilitlerini (Lock Table) RAM üzerinde yönetir.
    *   **Enqueue Replication Server 2 (ERS2):** Kilit tablosunu bellekten belleğe ikincil düğüme anlık replike eder. ASCS çökerse Pacemaker kümesi saniyeler içinde ASCS'yi ERS2'nin çalıştığı düğüm üzerinde ayağa kaldırır; kilitler kaybolmadığından kullanıcı oturumları kesilmez.
*   **PAS (Primary Application Server):** Sistemin ana uygulama sunucusudur; merkezi yönetimsel arka plan işlerini yürütür.
*   **AAS (Additional Application Server):** Sisteme yatay olarak eklenen uygulama sunucularıdır. Kullanıcı sayısına bağlı olarak onlarca AAS dinamik olarak kümeye dahil edilebilir.

### 3.2. Çekirdek Süreçler (Core OS Processes) ve İş İstekleri

```
+----------------------------------------------------------------------------------------------------+
| ABAP Instance (disp+work / Application Server Node)                                                |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | Internet Communication Manager (ICM - Process 'icman')                                       |  |
|  | - Epoll / Kqueue Tabanlı Asenkron Non-Blocking I/O Worker Havuzu                             |  |
|  | - HTTP(S), OData, WebSocket Terminasyonu, Thread Pooling                                     |  |
|  +----------------------------------------------+-----------------------------------------------+  |
|                                                 | (MPI - Memory Pipe Interface)                    |
|  +----------------------------------------------v-----------------------------------------------+  |
|  | ABAP Dispatcher (Process 'disp+work')                                                        |  |
|  | - Request Queue, Priority FIFO, Connection Multiplexing                                      |  |
|  +----------------------+-----------------------+-----------------------+-----------------------+  |
|                         |                       |                       |                          |
|  +----------------------v--+  +-----------------v--+  +-----------------v--+  +-----------------v--+  |
|  | DIA Work Process (Wp0)  |  | BTC Work Process   |  | UPD Work Process   |  | SPO Work Process   |  |
|  | [ABAP C++ Bytecode VM]  |  | [Batch Processing] |  | [Async Commit V1/V2|  | [Spool/Print Queue]|  |
|  +------------+------------+  +---------+----------+  +---------+----------+  +---------+----------+  |
|               |                         |                       |                       |             |
|  +------------v-------------------------v-----------------------v-----------------------v----------+  |
|  | SAP Extended Memory (EM) - OS POSIX Shared Memory (/dev/shm)                                    |  |
|  | [User Sessions, Contexts, Internal Tables (Pointers attached/detached per roundtrip)]          |  |
|  +-------------------------------------------------------------------------------------------------+  |
|  | Program Execute Area (PXA) & Global Buffer                                                      |  |
|  | [Compiled Bytecode, Shared Objects, Screen Painter Buffers, Database Table Buffers]            |  |
|  +-------------------------------------------------------------------------------------------------+  |
|  | Process-Local Heap (malloc memory)                                                              |  |
|  | [Fallback if EM exhausted -> Activates PRIV Mode (Process locked to single user)]              |  |
|  +-------------------------------------------------------------------------------------------------+  |
+-------------------------------------------------+--------------------------------------------------+
                                                  |
           +--------------------------------------+--------------------------------------+
           |                                                                             |
+----------v---------------------------------+         +---------------------------------v----------+
| Standalone Enqueue Server 2 (ENSA2)        |         | SAP Gateway (Process 'gwrd')               |
| - In-Memory Global Lock Table              |         | - RFC / CPIC Protokol Yöneticisi           |
| - Reader/Writer Lock Coordination          |         | - External System Connection Broker        |
+--------------------------------------------+         +--------------------------------------------+
```

*   **Internet Communication Manager (`icman`):** Dış dünyadan gelen HTTP(S), OData ve WebSocket isteklerini karşılar. Non-blocking I/O (`epoll`) kullanarak yükü **Memory Pipe Interface (MPI)** üzerinden paylaşılan belleğe yazar ve ABAP Dispatcher'a iletir.
*   **ABAP Dispatcher:** İstek kuyruğunu (FIFO) yönetir ve gelen yükü boşta olan iş süreçlerine (Work Process) dağıtır.
*   **Work Process Türleri:**
    *   **DIA (Dialog):** Kısa ömürlü kullanıcı ekran adımlarını ve REST/OData isteklerini yürütür (`rdisp/max_wprun_time` ile zaman aşımı denetimi yapılır).
    *   **BTC (Background):** Zaman aşımı sınırı olmayan toplu (batch) işleri yürütür.
    *   **UPD / UPD2 (Update):** Veritabanı tutarlılığı için iki aşamalı commit operasyonlarını asenkron tamamlar (V1 kritik işlemler, V2 istatistik/indeks güncellemeleri).
    *   **SPO (Spool):** Yazıcı ve belge dönüştürme (PDF vb.) kuyruklarını yönetir.
*   **SAP Gateway (`gwrd`):** RFC (Remote Function Call) ve CPIC protokollerini yönetir (`SM59` bağlantıları).

### 3.3. Bellek Alanları ve PRIV Mode Darboğazı

```
Request 1 (HTTP Get) ----> [ DIA Wp0 ] === Attach User Context from EM ===> Process -> Detach EM
Request 2 (HTTP Post) ---> [ DIA Wp3 ] === Attach Same User Context ===> Process -> Detach EM
```

*   **SAP Extended Memory (EM):** POSIX paylaşılan bellek segmentleridir (`/dev/shm`). Kullanıcının oturum değişkenleri ve dahili tabloları (internal tables) burada tutulur. İstek DIA sürecine geldiğinde bellek adreslenir (pointer attach); işlem bitince ayrılır (detach). Bu durumsuz (stateless) model sayesinde aynı kullanıcının sonraki adımı bambaşka bir DIA sürecinde yürütülebilir.
*   **Program Execute Area (PXA):** Derlenmiş ABAP bayt kodunun tutulduğu küresel paylaşılan bellek alanıdır.
*   **Process Local Heap ve PRIV Mode:**
    *   Bir kullanıcı oturumu tahsis edilen EM kotasını (`em/initial_size_MB`) aşarsa, iş süreci işletim sisteminden yerel bellek (`malloc`) almaya başlar.
    *   Bu duruma **PRIV Mode** denir. Bellek sürece yerel hale geldiğinden, süreç artık başka hiçbir kullanıcıya hizmet veremez ve oturum sonlanana kadar o kullanıcıya kilitlenir. Sistemde PRIV moduna giren süreç sayısının artması kuyrukların kilitlenmesine yol açar.

### 3.4. Uygulama Seviyesi Tablo Arabelleklemesi (Table Buffering)

HANA bellek içi olsa dahi her SQL çağrısı TCP soket gecikmesi ($0.2 - 0.5\text{ ms}$) yaratır. ABAP uygulama sunucusu sık okunan konfigürasyon tablolarını PAS/AAS'ın kendi **Shared Memory** katmanında arabelleğe alır:
*   **Single-Record Buffer:** Yalnızca birincil anahtarla çağrılan tekil satırlar belleğe alınır (`SELECT SINGLE`).
*   **Generic Buffer:** Tablo anahtarının belirli bir segmentine göre (örneğin Şirket Kodu `BUKRS`) kayıtlar gruplanarak saklanır.
*   **Full Table Buffer:** Tablonun tamamı (örneğin ülke kodları `T005`) PAS/AAS belleğine kopyalanır.
*   **Buffer Senkronizasyonu (`DDLOG`):** Bir sunucuda arabellekteki tablo güncellendiğinde veritabanındaki `DDLOG` tablosuna bir geçersiz kılma (invalidation) kaydı yazılır. Diğer sunucular bu kaydı okuyarak kendi yerel RAM tamponlarındaki ilgili veriyi siler ve veritabanından tekrar çeker.

### 3.5. İşlem Bütünlüğü: SAP LUW vs. Database LUW

Web veya ERP oturumlarında kullanıcının form doldurması dakikalar sürebilir. Veritabanının kendi işlem birimi (**Database LUW**) satır kilitlerini bu kadar süre tutamaz. Bu problemi çözmek için **SAP LUW** kullanılır:

```
KULLANICI DİYALOG ADIMI (Kullanıcı Form Dolduruyor - Dakikalar Sürebilir)
--------------------------------------------------------------------------------------
Ekran 1 (Giriş)  --> DIA WP1: Bellekte veri toplanır (SAP Lock alınır - ENSA2)
Ekran 2 (Kalem)  --> DIA WP4: Bellekte veri toplanır (DB bağlantısı boştadır)
Ekran 3 (Onay)   --> DIA WP2: 'COMMIT WORK' komutu tetiklenir.

GÜNCELLEME ADIMI (SAP LUW -> Database LUW Dönüşümü - Milisaniyeler İçinde Biter)
--------------------------------------------------------------------------------------
[ DIA Süreci ] 
      |  Fonksiyonları kuyruğa yazar: CALL FUNCTION ... IN UPDATE TASK
      |  VBDATA / VBMOD tablolarına geçici kayıtlar yazılır.
      v
[ COMMIT WORK ] 
      |  Tüm işlem paketi UPD (Update) Work Process'e fırlatılır.
      v
[ UPD Süreci ] 
      |  Tek bir gerçek Database LUW açar:
      |  1. BEGIN TRANSACTION (DB Level)
      |  2. INSERT INTO ACDOCA / MATDOC ...
      |  3. Veritabanı COMMIT tetiklenir.
      |  4. ENSA2 üzerindeki SAP kilitleri serbest bırakılır.
      v
[ DB COMMIT ] (İşlem kalıcı hale gelir)
```

---

## 4. ERP Evrimi: ECC'den S/4HANA'ya Veri Modeli ve Geçiş Stratejileri

| Kriter | SAP ECC 6.0 (Legacy) | SAP S/4HANA (Modern) |
| :--- | :--- | :--- |
| **Veritabanı Desteği** | Herhangi bir DB (Oracle, MSSQL, DB2, MaxDB). | Yalnızca SAP HANA (In-Memory). |
| **Finansal Tablo Mimarisi** | Ayrık indeks ve toplama tabloları (`BSIS`, `BSAS`, `GLT0`, `BSEG`). | Sadeleştirilmiş tek evrensel finans tablosu (**`ACDOCA`**). |
| **Lojistik Tablo Mimarisi** | Ayrık stok ve malzeme hareket tabloları (`MKPF`, `MSEG`, `MBEW`, `MARC`). | Sadeleştirilmiş tek evrensel malzeme tablosu (**`MATDOC`**). |
| **İş Ortağı Mimarisi** | Müşteri (`KNA1`) ve Satıcı (`LFA1`) ayrık tabloları. | Zorunlu **Business Partner (BP)** ve CVI entegrasyonu. |
| **Analitik & İşlem Ayrımı** | OLTP ve OLAP ayrıdır; raporlar için veri BW'ya aktarılır. | **HTAP**; operasyonel tablolardan Core Data Services (CDS) ile anlık analiz. |
| **Kullanıcı Arayüzü** | SAP GUI (Özel binary DIAG protokolü). | SAP Fiori (Web tabanlı UI5, OData üzerinden REST/JSON). |

### 4.1. Veri Sadeleştirmesi: Universal Journal (`ACDOCA` & `MATDOC`)

*   **`ACDOCA` (Universal Journal):** Genel muhasebe (FI-GL), maliyet muhasebesi (CO), varlık yönetimi (AA) ve kârlılık analizini (CO-PA) tek bir tabloda yüzlerce sütunla birleştirir. Eski toplama ve indeks tabloları kaldırılarak veri tabanı seviyesindeki kilit çakışmaları engellenmiştir.
*   **`MATDOC` (Universal Material Document):** ECC'deki malzeme belgesi başlığı (`MKPF`) ve kalemleri (`MSEG`) ile değerleme tabloları (`MBEW`) tek bir tabloda toplanmıştır. Envanter sayımları ve stok hareketleri anlık olarak bu tablodan hesaplanır.
*   **Business Partner (BP) ve CVI (Customer Vendor Integration):** ECC'deki müşteri ve satıcı ayrımı kaldırılmıştır. Artık her tüzel veya gerçek kişilik tek bir **Business Partner** nesnesidir ve roller (Müşteri, Satıcı, Banka) bu nesneye bağlanır.

### 4.2. S/4HANA Geçiş Yolları (Migration Paths)

```
                  ┌─────────────────────────────────────────────────────────┐
                  │          S/4HANA DÖNÜŞÜM METODOLOJİLERİ                 │
                  └────────────────────────────┬────────────────────────────┘
         ┌─────────────────────────────────────┼─────────────────────────────────────┐
         ▼                                     ▼                                     ▼
┌──────────────────┐                  ┌──────────────────┐                  ┌──────────────────┐
│   GREENFIELD     │                  │    BROWNFIELD    │                  │    BLUEFIELD     │
│ (Yeni Kurulum)   │                  │(Sistem Dönüşümü) │                  │(Seçici Veri Göçü)│
├──────────────────┤                  ├──────────────────┤                  ├──────────────────┤
│• Sıfır miras kod │                  │• SUM DMO ile tek │                  │• Eski sistemden  │
│• Temiz Çekirdek  │                  │  adımda geçiş    │                  │  yalnızca istenen│
│• Yüksek iş süreci│                  │• Tarihsel veri   │                  │  şirket/yıllar   │
│  değişimi        │                  │  bütünüyle korunur│                 │  ayrıştırılır    │
└──────────────────┘                  └──────────────────┘                  └──────────────────┘
```

1.  **Greenfield (Yeni Kurulum):** Mevcut ERP geçmişi geride bırakılır; SAP Best Practice süreçleriyle sistem sıfırdan kurulur. Clean Core için en temiz yaklaşımdır.
2.  **Brownfield (Sistem Dönüşümü - System Conversion):** Mevcut ECC sistemi, özelleştirmeleri ve tarihsel verisiyle birlikte S/4HANA'ya dönüştürülür.
    *   **SUM DMO (Software Update Manager - Database Migration Option):** Veritabanı geçişini (AnyDB $\rightarrow$ HANA), yazılım sürüm yükseltmesini (ECC $\rightarrow$ S/4HANA) ve Unicode dönüşümünü tek bir kesinti (downtime) penceresinde gerçekleştiren çekirdek araçtır.
3.  **Bluefield (Seçici Veri Geçişi - Selective Data Transition):** Birden fazla ECC sistemini tek bir S/4HANA'da konsolide etmek veya yalnızca son 3 yılın verilerini alıp kalanını arşivlemek için özel araçlarla (SNP CrystalBridge vb.) yapılan geçiştir.

### 4.3. Sürüm Takvimi ve Destek Modeli

S/4HANA 2023 sürümüyle birlikte SAP sürüm modelini güncellemiştir:
*   **2 Yılda Bir Ana Sürüm:** (Örn: 2023, 2025, 2027).
*   **5 Yıl Standart Destek:** Her ana sürüm 5 yıl ana destek kapsamındadır.
*   **Feature Pack Stack (FPS):** İlk iki yıl boyunca yılda iki kez yeni fonksiyonel paketler (FPS01, FPS02) yayınlanır; ardından hata düzeltme paketlerine (SPS) geçilir.

---

## 5. Dağıtım Modelleri ve Kurumsal Bulut Stratejisi (RISE & GROW)

```
[ Geleneksel Özgürlük / Yüksek Bakım Maliyeti ]  <=================>  [ Standardizasyon / Düşük TCO / Temiz Çekirdek ]
                     
         On-Premise                      S/4HANA Private Cloud                 S/4HANA Public Cloud
   +--------------------+               +--------------------+               +--------------------+
   |   Müşteri Altyapı  |               |   Hyperscaler /    |               |  SAP Tarafından    |
   | (Data Center/IaaS) |               |  SAP Managed Cloud |               | Paylaşılan SaaS    |
   +--------------------+               +--------------------+               +--------------------+
   | Tam OS & DB Erişimi|               | Yönetilen IaaS/DB  |               | Sıfır Altyapı/OS   |
   | (Linux root, HANA) |               | (Sınırlı OS yetkisi|               | Erişimi (Tam SaaS) |
   +--------------------+               +--------------------+               +--------------------+
   | Klasik ABAP (Z*)   |               | Klasik + ABAP Cloud|               | Yalnızca ABAP Cloud|
   | & Tablo Modifikas. |               | (Clean Core Opsiy.)|               | & Clean Core       |
   +--------------------+               +--------------------+               +--------------------+
   | Yükseltme: İsteğe  |               | Yükseltme: Müşteri |               | Yükseltme: Zorunlu |
   | bağlı (Yıllarca yok|               | planlı (Yıllık)    |               | 6 ayda bir (Oto.)  |
   +--------------------+               +--------------------+               +--------------------+
```

1.  **SAP S/4HANA On-Premise:** Sunucu donanımı ve veri merkezi yönetimi müşteridedir. İşletim sistemi (`root`) ve veritabanı (`SYSTEM`) tam erişime açıktır. Kaynak kodlar modifiye edilebilir; ancak bu durum yükseltme maliyetlerini artırır.
2.  **SAP S/4HANA Cloud, Private Edition (PCE / RISE with SAP):** Tek kiracılı (single-tenant) yapıdır. Altyapı Hyperscaler (AWS/Azure/GCP) üzerinde SAP tarafından yönetilir. Klasik ABAP desteği korunur; ancak müşteriler Clean Core prensiplerine yönlendirilir. Altyapı ve veritabanı işletimi SAP'nin SLA sözleşmesi altındadır.
3.  **SAP S/4HANA Cloud, Public Edition (GROW with SAP):** Tam bir çok kiracılı SaaS (Software-as-a-Service) modelidir. İşletim sistemi veya veritabanı erişimi yoktur. `SAP GUI` kullanılamaz; geliştirme yalnızca bulut standartlarındaki API ve araçlarla yapılabilir. 6 ayda bir otomatik olarak güncellenir.

---

## 6. "Clean Core" Paradigması ve Modern Yazılım Geliştirme (RAP & ABAP Cloud)

Clean Core; ERP çekirdeğini standart tutarak modifikasyonlardan arındırmayı, yükseltmeleri (upgrades) kesintisiz hale getirmeyi ve iş mantığını dış sistemlerle gevşek bağlı (loosely coupled) şekilde genişletmeyi hedefleyen yazılım mimarisi disiplinidir.

```
               ┌────────────────────────────────────────────────────────┐
               │              CLEAN CORE'UN 5 BOYUTU                    │
               └───────────────────────────┬────────────────────────────┘
      ┌──────────────┬─────────────┬───────┴─────┬──────────────┬──────────────┐
      ▼              ▼             ▼             ▼              ▼              ▼
┌───────────┐  ┌───────────┐ ┌───────────┐ ┌───────────┐  ┌───────────┐  ┌───────────┐
│Extensibil.│  │ Processes │ │   Data    │ │Integration│  │Operations │  │  Erişim   │
│ABAP Cloud │  │ Standard  │ │Yönetişim, │ │ Released  │  │ Cloud ALM │  │Sözleşmesi │
│Side-by-Side  │ Best Pract│ │ Doğruluk, │ │  APIs,    │  │ Otomatik  │  │ C1, C2, C3│
│    (BTP)  │  │ & No-Modif│ │ Arşivleme │ │Event-Driv.│  │  Testler  │  │ Sözleşme  │
└───────────┘  └───────────┘ └───────────┘ └───────────┘  └───────────┘  └───────────┘
```

### 6.1. Clean Core 5 Boyutu
1.  **Extensibility (Genişletilebilirlik):** Standart kod tabanını doğrudan değiştirmemek (`SE38` modifikasyon yasağı); yalnızca ABAP Cloud ve BTP kullanmak.
2.  **Processes (Süreçler):** Mümkün olduğunca SAP Best Practice iş süreçlerine sadık kalmak.
3.  **Data (Veri):** Veri kirliliğini önlemek, yaşam döngüsü kurallarını (ILM) uygulayarak veri tabanı hacmini optimize etmek.
4.  **Integration (Entegrasyon):** Noktadan noktaya (point-to-point) veritabanı bağlantıları yerine yalnızca kararlı API'lar ve asenkron olaylar (Event-Driven Architecture) kullanmak.
5.  **Operations (Operasyon):** Yama ve sürüm yükseltmelerini otomatik test araçları ve SAP Cloud ALM ile yönetmek.

### 6.2. Extensibility Modelleri ve Yayınlanmış Sözleşmeler (Release Contracts)

```
                                  [ Dış İstemciler / Fiori UI ]
                                                |
                                                v
+-----------------------------------------------------------------------------------------------+
| S/4HANA Core                                                                                  |
|                                                                                               |
|  +-----------------------------------------------------------------------------------------+  |
|  | Key User Extensibility: Düşük-Kod / UI Alan Ekleme (No-Code App Engine)                 |  |
|  +-----------------------------------------------------------------------------------------+  |
|                                                                                               |
|  +-----------------------------------------------------------------------------------------+  |
|  | Developer Extensibility (ABAP Cloud On-Stack):                                          |  |
|  | Sadece kararlı 'Released API' (C1 Sözleşmeleri) kullanabilen kısıtlı ABAP çalışma zamanı|  |
|  +--------------------------------------------+--------------------------------------------+  |
|                                               |                                               |
|  +--------------------------------------------v--------------------------------------------+  |
|  | Standart Çekirdek (Clean Core): Doğrudan erişime kapalı standart veri yapıları         |  |
|  +-----------------------------------------------------------------------------------------+  |
+-----------------------------------------------+-----------------------------------------------+
                                                |
                                (OData / REST / Event Mesh)
                                                |
                                                v
+-----------------------------------------------------------------------------------------------+
| SAP BTP (Side-by-Side Extensibility):                                                         |
| Çekirdekten bağımsız mikroservisler (Node.js, Java, Python, Go, Cloud Foundry, K8s Kyma)      |
+-----------------------------------------------------------------------------------------------+
```

*   **Key User Extensibility:** Kod yazmadan, Fiori arayüzünden yeni alan ekleme ve basit doğrulama kuralı tanımlama.
*   **On-Stack Developer Extensibility (ABAP Cloud):** S/4HANA üzerinde kod yazılır; ancak yalnızca **Eclipse ADT (ABAP Development Tools)** kullanılır. Klasik dynpro ekranları ve doğrudan standart tablolara SQL atma yasaktır.
*   **Side-by-Side Extensibility (SAP BTP):** Çekirdekten tamamen bağımsız mikroservisler (Node.js, Java, Python).
*   **Sözleşme Kapsamları (Release Contracts):**
    *   **`C1` (System-Internal Use):** S/4HANA içerisinde ABAP Cloud geliştiricilerinin doğrudan tüketebileceği kararlı CDS View ve sınıflar.
    *   **`C2` (Cross-System Use):** Sistemler arası BTP veya dış SaaS tüketicileri için yayınlanan kararlı OData/REST servisleri.
    *   **`C3` (Enhancement Stability):** Standart koda müdahale etmeden araya giren BAdI genişletme noktaları.

### 6.3. ABAP RESTful Application Programming Model (RAP) Mimarisi

RAP, modern ABAP Cloud mimarisinin resmi geliştirme çerçevesidir:

```
+---------------------------------------------------------------------------------+
| PRESENTATION LAYER: SAP Fiori Elements (UI5) / OData v4 Consumer                |
+---------------------------------------------------------------------------------+
                                      ^
                                      | (OData v4 REST Protocol)
+-------------------------------------v-------------------------------------------+
| SERVICE BINDING: Protokol Eşleme (OData V4 - UI, Web API)                       |
+---------------------------------------------------------------------------------+
| SERVICE DEFINITION: Tüketilecek Varlıkların Kapsam Tanımı                       |
+---------------------------------------------------------------------------------+
| PROJECTION LAYER: Consumption Views (C_BookingTP) + UI Annotations              |
+---------------------------------------------------------------------------------+
| BUSINESS OBJECT LAYER:                                                          |
| - Data Definition: Interface CDS Views (I_BookingTP)                            |
| - Behavior Definition (BDEF): Fiiller (Create, Update, Delete, Actions, Draft)  |
| - Behavior Pool: Yerel C++ ABAP Sınıfları (Validations, Determinations)         |
+---------------------------------------------------------------------------------+
| DATABASE: SAP HANA In-Memory (ACDOCA / Custom Tables)                           |
+---------------------------------------------------------------------------------+
```

#### RAP Veri Tanımı Örneği (CDS View Entity)
```sql
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: 'Siparis Interface View'
define root view entity I_CustomOrder
  as select from zorder_table
{
  key order_id      as OrderID,
      customer_id   as CustomerID,
      total_amount  as TotalAmount,
      currency_code as CurrencyCode,
      order_status  as OrderStatus
}
```

#### RAP Davranış Tanımı Örneği (Behavior Definition - `BDEF`)
```abap
managed implementation in class zbp_i_customorder unique;
strict ( 2 );

define behavior for I_CustomOrder alias Order
persistent table zorder_table
lock master
authorization master ( instance )
{
  create;
  update;
  delete;

  field ( readonly ) OrderID;
  field ( mandatory ) CustomerID, TotalAmount;

  action ( features : instance ) approveOrder result [1] $self;
  validation validateAmount on save { create; field TotalAmount; }
}
```

---

## 7. Ağ Güvenliği, Entegrasyon ve Kimlik Yönetimi (Cloud Connector, IAS/IPS)

### 7.1. SAP Cloud Connector: Ters Tünel (Reverse Invocation) Mimarisi

Kurum içi (on-premise) kurumsal ağlar ile genel bulut (SAP BTP / SaaS) arasındaki entegrasyon, güvenlik duvarı portlarının dış dünyaya kapatılması zorunluluğu nedeniyle **Ters Tünelleme (Reverse Invocation)** mimarisine dayanır.

```
+------------------------------------+                      +---------------------------------------+
| SAP BTP Environment (Public Cloud) |                      | Customer On-Premise Network (DMZ/LAN) |
|                                    |                      |                                       |
|  +------------------------------+  |                      |  +---------------------------------+  |
|  | SAP Connectivity Service     |  |                      |  | SAP Cloud Connector (SCC)       |  |
|  | - Multiplexed Proxy Gateway  |  |                      |  | - Lightweight JVM Daemon        |  |
|  +--------------+---------------+  |                      |  +----------------+----------------+  |
|                 |                  |                      |                   |                   |
|                 |  mTLS Tunnel     |   Outbound 443 Only  |                   |                   |
|                 |  (RFC 2818/8446) | <=================== |                   |                   |
|                 |  (Persistent)    |   (No Inbound Ports) |                   |                   |
|                 |                  |                      |                   |                   |
|  +--------------v---------------+  |                      |  +----------------v----------------+  |
|  | Subscribed Cloud Services    |  |                      |  | Internal Backend Systems        |  |
|  | (Cloud Foundry, Kyma, IAS)   |  |                      |  | (S/4HANA, RFC Gateway, REST)    |  |
|  +------------------------------+  |                      |  +---------------------------------+  |
+------------------------------------+                      +---------------------------------------+
```

*   **Sıfır Gelen Port Kuralı:** Kurumsal güvenlik duvarında içeriye doğru (inbound) hiçbir port açılmaz. Cloud Connector, kurum içinden BTP Connectivity Service'e doğru giden (outbound) yönde `443` portu üzerinden kalıcı bir TLS tüneli açar.
*   **Çift Yönlü mTLS (Mutual TLS):** Tünel kurulurken istemci ve sunucu karşılıklı olarak dijital sertifikalarını doğrular. Buluttan kurum içine gelen istekler, bu açık tünel içinden tersine akıtılır (multiplexed reverse tunnel).
*   **Virtual-to-Internal URL Maskeleme:** Kurum içi IP adresleri gizlenir (Örn: Sanal `s4core.corp.local:443` adresi iç ağda `10.20.4.50:44300` adresine yönlendirilir). Sadece onaylanan API uç noktalarına izin verilir.

### 7.2. Principal Propagation (Kriptografik Kimlik Yayılımı)

Bulut üzerindeki kullanıcının kimliğinin şifre taşınmadan doğrudan kurum içi ABAP sistemine aktarılmasıdır:

```
[ BTP Cloud App ] 
      |  (Bearer JWT Token: john.doe@company.com)
      v
[ Connectivity Service ] 
      |  (Encrypted Tunnel Payload + User Assertion)
      v
[ Cloud Connector ] 
      |  (X.509 Short-Lived User Certificate Generation via Local CA)
      v
[ S/4HANA ICM / SNC Gateway ] (Doğrudan 'JDOE' kullanıcısı bağlamında oturum açılır)
```

1.  Kullanıcı BTP uygulamasında oturum açar (JWT Token üretilir).
2.  BTP servisi isteği SCC'ye iletirken kullanıcı kimliğini başlığa ekler.
3.  Cloud Connector, kendi üzerindeki yerel Sertifika Otoritesi (Local CA) ile ilgili kullanıcı adına **kısa ömürlü (1-2 dakikalık) bir X.509 istemci sertifikası** üretir.
4.  S/4HANA sistemi bu sertifikayı doğrular ve `CERTRULE` / `USRACEXT` tablosundaki kurallara göre isteği doğrudan o kullanıcının ERP hesabı bağlamında çalıştırır.

### 7.3. Kimlik Yaşam Döngüsü: IAS, IPS ve JML Entegrasyonu

```
+----------------------------------------------------------------------------------------------------+
| Kimlik Sağlayıcı (IdP - Microsoft Entra ID / Okta)                                                |
+-------------------------------------------------+--------------------------------------------------+
                                                  | SAML 2.0 / OpenID Connect Federasyonu
                                                  v
+----------------------------------------------------------------------------------------------------+
| SAP Cloud Identity Services                                                                        |
|                                                                                                    |
|  +----------------------------------------------------------------------------------------------+  |
|  | SAP Identity Authentication Service (IAS)                                                    |  |
|  | - Single Sign-On (SSO) Proxy / Identity Broker                                               |  |
|  | - Multi-Factor Authentication (MFA), Risk-Based Conditional Access Policies                      |  |
|  +----------------------------------------------+-----------------------------------------------+  |
|                                                 |                                                  |
|  +----------------------------------------------v-----------------------------------------------+  |
|  | SAP Identity Provisioning Service (IPS)                                                      |  |
|  | - JSON-to-JSON SCIM 2.0 Veri Dönüştürücüsü (Attribute Mapping Engine)                         |  |
|  | - Olay Güdümlü / Zamanlanmış Eşitleme (Delta Reading via State Storage)                      |  |
|  +----------------------------------------------+-----------------------------------------------+  |
+-------------------------------------------------+--------------------------------------------------+
                                                  |
           +--------------------------------------+--------------------------------------+
           | SCIM 2.0 REST API                                   | SCIM / OData Connector
           v                                                     v
+------------------------------------+                 +------------------------------------+
| SAP BTP Subaccounts                |                 | S/4HANA On-Prem / Private Cloud    |
| - Shadow User Provisioning         |                 | - Identity Management (SU01)       |
| - Role Collection Atamaları        |                 | - Business Partner (BP) Creation   |
+------------------------------------+                 +------------------------------------+
```

*   **IAS (Identity Authentication Service):** Kurumsal IdP (Microsoft Entra ID, Okta vb.) ile SAP sistemleri arasında kimlik doğrulama köprüsüdür (SAML 2.0 / OIDC).
*   **IPS (Identity Provisioning Service):** SCIM 2.0 protokolü üzerinden kullanıcıları ve yetki gruplarını sistemler arasında senkronize eder.

#### SCIM Tabanlı Dönüştürme Mantığı (Transformation Engine)

IPS, SCIM 2.0 standardını kullanır. Kaynak sistemden (ör. SuccessFactors veya Entra ID) çekilen JSON nesneleri, deklaratif JSON şablonlarıyla hedef sistem şemasına dönüştürülür:

```json
{
  "user": {
    "mappings": [
      {
        "sourcePath": "$.emails[0].value",
        "targetPath": "$.userName",
        "condition": "$.emails[0].primary == true"
      },
      {
        "sourcePath": "$.name.familyName",
        "targetPath": "$.name.familyName"
      },
      {
        "constant": "INTERNAL_EMPLOYEE",
        "targetPath": "$.userType"
      }
    ]
  }
}
```

*   **JML Süreç Akışı (Joiner - Mover - Leaver):**
    *   **Joiner:** İK sisteminde (SuccessFactors) yeni personel açıldığında IPS anlık olarak IAS'ta kimliği oluşturur, BTP'ye rollerini yazar ve S/4HANA'da `BAPI_USER_CREATE1` ile kullanıcıyı üretir.
    *   **Mover:** Departman değişiminde eski `Role Collections` silinir, yenileri bağlanır.
    *   **Leaver:** Personel ayrıldığında IAS hesabı pasife çekilir; tüm açık SSO oturumları anında sonlandırılarak sistem erişimleri kapatılır.

---

## 8. SAP BTP, Veri Kumaşı ve Genişletilmiş SaaS Portföyü

### 8.1. SAP BTP Runtimes ve Integration Suite

```
+---------------------------------------------------------------------------------+
| SAP Business Technology Platform (BTP)                                          |
|                                                                                 |
|  +---------------------------------------------------------------------------+  |
|  | Runtime Ortamları:                                                        |  |
|  | - Cloud Foundry (Buildpack tabanlı PaaS: Node.js, Java, Python, Go)       |  |
|  | - Kyma Runtime (Managed Kubernetes / K8s & Istio tabanlı mikroservisler)  |  |
|  | - ABAP Cloud Environment ("Steampunk" - Buluttaki bağımsız ABAP VM)      |  |
|  +---------------------------------------------------------------------------+  |
|                                                                                 |
|  +---------------------------------------------------------------------------+  |
|  | Entegrasyon ve Bağlantı (Integration Suite):                              |  |
|  | - Cloud Integration (CPI): Enterprise Integration Patterns (EIP)          |  |
|  | - Event Mesh / Advanced Event Mesh: AMQP / MQTT tabanlı asenkron pub/sub  |  |
|  | - API Management: Rate limiting, API Gateway, JWT doğrulama               |  |
|  +---------------------------------------------------------------------------+  |
|                                                                                 |
|  +---------------------------------------------------------------------------+  |
|  | Kimlik & Güvenlik (Cloud Identity Services):                              |  |
|  | - IAS (Identity Authentication Service): SAML 2.0 / OIDC IdP Broker       |  |
|  | - IPS (Identity Provisioning Service): SCIM tabanlı senkronizasyon        |  |
|  +---------------------------------------------------------------------------+  |
+---------------------------------------^-----------------------------------------+
                                        |
                            (Mutual TLS Ters Tünel)
                                        |
+---------------------------------------v-----------------------------------------+
| SAP Cloud Connector (Şirket İçi DMZ'de Çalışan Reverse Proxy Ajanı)             |
+---------------------------------------------------------------------------------+
                                        |
                      [ On-Premise S/4HANA / RFC / BAPI ]
```

*   **Cloud Foundry:** Standart Buildpack tabanlı mikroservis platformu (Node.js, Java, Python, Go).
*   **Kyma Runtime:** Kubernetes ve Istio servis ağı (Service Mesh) tabanlı, konteynerleştirilmiş bulut yerel (cloud-native) mikroservis ortamı.
*   **ABAP Cloud Environment ("Steampunk"):** BTP üzerinde bağımsız çalışan, SAP HANA Cloud tabanlı izole ABAP sanal makinesi.
*   **Integration Suite:**
    *   *Cloud Integration (CPI):* Kurumsal entegrasyon desenlerini (EIP - Splitter, Gather, Router) yürüten ESB katmanı.
    *   *Event Mesh / Advanced Event Mesh:* AMQP ve MQTT tabanlı olay güdümlü (event-driven pub/sub) mesajlaşma omurgası.

### 8.2. Süreç Dönüşümü ve Kurumsal Mimari: Signavio & LeanIX
*   **SAP Signavio (Business Process Transformation):**
    *   *Process Insights & Intelligence:* S/4HANA üzerindeki işlem loglarını tarayarak süreç darboğazlarını (Process Mining) ortaya çıkarır.
    *   *Process Manager:* Şirketin iş süreçlerini BPMN 2.0 standardında modeller.
*   **SAP LeanIX (Enterprise Architecture Management):**
    *   Şirketin tüm BT uygulama envanterini, veri akışlarını, kullanılan teknolojileri ve bunların yaşam döngülerini görselleştiren kurumsal mimari platformudur.

### 8.3. Modern Veri Kumaşı: SAP Datasphere ve Analytics Cloud (SAC)
*   **SAP Datasphere (Business Data Fabric):**
    *   Eski SAP BW mimarisinin bulut tabanlı ardılıdır. Veriyi fiziksel olarak taşımadan yerinde sorgulayan **Veri Sanallaştırması (Data Virtualization)** yeteneğine sahiptir.
    *   *Semantik Katman:* S/4HANA CDS modellerindeki iş mantığını (para birimi dönüşümleri, hiyerarşiler) bozmadan analitik katmana taşır.
    *   *Spaces:* Farklı departmanların (Finans, Satış) izole veri çalışma alanları oluşturmasını sağlar.
*   **SAP Analytics Cloud (SAC):** İş zekası (BI), kurumsal bütçe planlama ve tahmine dayalı analitiği tek platformda birleştiren SaaS çözümüdür.

### 8.4. Master Data Integration (MDI) ve One Domain Model (ODM)
*   Farklı SaaS uygulamaları (Ariba, Concur, SuccessFactors, S/4HANA) kendi veri modellerine sahiptir.
*   **One Domain Model (ODM):** Tüm SAP ekosistemi için tanımlanmış ortak veri standardıdır (JSON-LD şemaları).
*   **SAP Master Data Integration (MDI):** ODM formatını kullanarak şirket genelindeki ana veriyi (Müşteri, Tedarikçi, Masraf Merkezi, Çalışan) merkezi bir olay kuyruğu üzerinden otomatik senkronize eden BTP servisidir.

### 8.5. Çevre SaaS Portföyü Entegrasyon Matrisi

```
                             +------------------------+
                             |   SAP S/4HANA Core     |
                             +-----------+------------+
                                         |
     +-------------------+---------------+-------------------+--------------------+
     |                   |                                   |                    |
(OData/IPS)         (cXML/BTP Integration)            (REST APIs)           (HANA Engine)
     |                   |                                   |                    |
     v                   v                                   v                    v
+----+---------+   +-----+--------+                    +-----+--------+     +-----+--------+
|   SAP        |   |   SAP        |                    |   SAP        |     |   SAP        |
|SuccessFactors|   |  Ariba       |                    |  Concur      |     |   IBP        |
|   (HR/HCM)   |   | (Satınalma)  |                    | (Masraf/Sev.)|     | (Tedarik Pl.)|
+--------------+   +--------------+                    +--------------+     +--------------+
```

*   **SAP SuccessFactors:** İnsan kaynakları yönetimi. Çalışan verileri SCIM/IPS ve MDI üzerinden S/4HANA Business Partner kayıtlarına aktarılır.
*   **SAP Ariba:** B2B satınalma ve tedarikçi ağı. Sipariş ve faturalar **cXML** formatında Integration Suite üzerinden malzeme yönetimi (MM) modülüyle çift yönlü akar.
*   **SAP Concur:** Masraf ve seyahat yönetimi. Harcama kayıtları REST API'lar ile S/4HANA finansal belgelerine (`ACDOCA`) dönüştürülür.
*   **SAP IBP (Integrated Business Planning):** Bellek içi tahminleme algoritmalarıyla çalışan gelişmiş tedarik zinciri planlama çözümüdür.
*   **SAP Fieldglass:** Dış kaynak, sözleşmeli personel ve hizmet tedariki yönetim platformudur.

---

## 9. Kurumsal Yapay Zeka Katmanı: SAP Business AI, Joule ve Vektör Motoru

SAP'nin yapay zeka stratejisi, genel amaçlı modelleri kurumsal iş bağlamı (enterprise context) ve kullanıcı yetkileriyle birleştiren **SAP Business AI** mimarisine dayanır.

```
+-----------------------------------------------------------------------------------+
| SAP BUSINESS AI MİMARİSİ                                                          |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | SAP JOULE (Kurumsal Yapay Zeka Asistanı)                                    |  |
|  | - Web Components / Fiori Entegrasyonu                                       |  |
|  | - Context Grounding: RAG Yöntemiyle SAP İş Metadata ve Yetkilerini Eşleme   |  |
|  | - Action Execution: OData/RAP Servislerini Otonom Çalıştırma                |  |
|  +--------------------------------------+--------------------------------------+  |
|                                         |                                         |
|  +--------------------------------------v--------------------------------------+  |
|  | SAP BTP AI CORE & GENERATIVE AI HUB                                         |  |
|  | - Foundation Modeller: OpenAI (GPT-4o), Anthropic (Claude), Mistral, vb.    |  |
|  | - Resource Groups ile Çok Kiracılı İzolasyon                                |  |
|  | - Veri Gizliliği: Sıfır Müşteri Verisi Saklama (Zero Data Retention SLA)    |  |
|  +--------------------------------------+--------------------------------------+  |
|                                         |                                         |
|  +--------------------------------------v--------------------------------------+  |
|  | SAP HANA CLOUD VECTOR ENGINE                                                |  |
|  | - İş belgelerinin (PDF, fatura, kontrat) gömme (embedding) vektörleri      |  |
|  | - L2 Mesafe / Cosine Similarity Aramaları ile Hibrit İşlem + Vektör Sorgusu  |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

### 9.1. SAP Joule: Bağlam Farkındalıklı Kurumsal Asistan
*   **Context Grounding:** Kullanıcı Joule'e bir soru sorduğunda (Örn: *"Bu ayki bütçe aşımına hangi satınalma siparişleri neden oldu?"*), sistem isteği doğrudan LLM'e göndermez.
*   Önce kullanıcının S/4HANA yetkilerini (`PFCG`) kontrol eder. Yetkisi dahilindeki verileri SAP iş grafından (Business Data Web) toplar, RAG (Retrieval-Augmented Generation) yöntemiyle bağlama ekler ve prompt'u zenginleştirerek güvenli modeller üzerinden yanıt üretir.
*   **Action Execution:** Joule sadece bilgi vermez; kullanıcının onaylaması durumunda arka plandaki RAP ve OData servislerini tetikleyerek işlem (Örn: Siparişi onaylama veya bloke koyma) yürütebilir.

### 9.2. SAP AI Core ve Generative AI Hub
*   **Generative AI Hub:** Kurumların tek bir kurumsal API üzerinden lider büyük dil modellerine (OpenAI GPT-4o, Anthropic Claude, Mistral Large, Aleph Alpha) erişmesini sağlar.
*   **Veri Gizliliği ve Güvenlik:** Müşteri verileri modellerin yeniden eğitilmesinde kesinlikle kullanılmaz; üçüncü parti model sağlayıcılarıyla **Zero Data Retention** (Veri Saklamama) sözleşmeleriyle çalışır.
*   **Resource Groups:** BTP AI Core üzerinde farklı iş birimlerinin kota, harcama ve model erişimlerini birbirinden izole eder.

### 9.3. SAP HANA Cloud Vector Engine
*   Ayrı bir harici vektör veritabanı (Pinecone, Milvus vb.) kurma ihtiyacını ortadan kaldırır.
*   Kurumsal belgelerin (PDF, fatura, şartname) metin gömme (text embedding) vektörleri doğrudan HANA tablolarında `REAL_VECTOR` veri tipinde saklanır.
*   **Hibrit Sorgulama:** Aynı SQL sorgusu içinde hem ilişkisel filtreleme (`WHERE SatisTemsilcisi = 'Ahmet'`) hem de anlamsal vektör benzerliği araması (`COSINE_SIMILARITY` veya `L2DISTANCE`) milisaniyeler seviyesinde tek işlemde yürütülür.

---

## 10. Uygulama Yaşam Döngüsü (ALM), DevOps ve Yetkilendirme/Yönetişim (GRC)

### 10.1. Yetkilendirme Mimarisi (PFCG Internals)

ABAP sistemlerinde yetkilendirme işlemleri menü seviyesinde değil, çekirdek seviyesinde kontrol edilen **Yetki Nesneleri (Authorization Objects)** üzerinden yürütülür:

```
[ Kullanıcı (SU01) ] ---> [ Rol (PFCG) ] ---> [ Yetki Nesnesi (Auth Object) ] ---> [ Alanlar & Aktiviteler ]
                                                    (Örn: F_BKPF_BUK)                - BUKRS: 1000, 2000
                                                                                     - ACTVT: 01 (Create), 03 (Display)
```

*   **Yetki Nesnesi Anatomisi:** En fazla 10 yetki alanından (Authorization Field) oluşur. İşlem kodları (`SU24` tablosu) yürütüldüğünde, ABAP çekirdeği `AUTHORITY-CHECK OBJECT 'F_BKPF_BUK' FIELD 'BUKRS' FIELD 'ACTVT'` komutunu çağırır.
*   **Rol Hiyerarşisi:**
    *   *Tekil Roller (Single Roles):* Yetki nesnelerini doğrudan barındırır.
    *   *Türetilmiş Roller (Derived Roles):* Ana rolün menü ve yetki şablonunu miras alır; ancak organizasyonel alan değerleri (Şirket Kodu, Satış Örgütü) şube veya fabrika bazında farklılaştırılır.
    *   *Birleşik Roller (Composite Roles):* Kullanıcılara tek tek rol atamak yerine mantıksal olarak gruplanmış tek bir rol paketi sunar.
*   **`SU24` ve `SU25` Optimizasyonu:** `SU24`, hangi işlem kodunun hangi yetki nesnelerini denetleyeceğini saklar. Sistem yükseltmelerinde `SU25` kullanılarak SAP'nin yeni getirdiği yetki kontrolleri kurumsal yapıya uyarlanır.

### 10.2. Kurumsal Risk ve Yönetişim: SAP GRC vs. SAP Cloud IAG
*   **SAP GRC Access Control:** On-premise sistemlerde Görevler Ayrılığı (Segregation of Duties - SoD) ihlallerini denetler. (Örn: Fatura girişi yapan bir kullanıcının aynı zamanda ödeme onaylama yetkisine sahip olmasını engeller).
    *   *Emergency Access Management (EAM / Firefighter):* Üretim ortamında acil müdahaleler için yetkisi geçici olarak yükseltilmiş özel süper kullanıcı oturumları açar ve yapılan tüm işlemleri denetim loglarına kaydeder.
*   **SAP Cloud Identity Access Governance (Cloud IAG):** GRC mantığını BTP, SuccessFactors, Ariba ve S/4HANA Cloud gibi hibrit bulut ortamlarına taşıyan, yapay zeka destekli rol madenciliği yapabilen SaaS çözümüdür.

### 10.3. Yazılım Taşıma Yönetimi: STMS'ten gCTS'e Geçiş

```
KLASİK TAŞIMA (STMS):
[ DEV ] --- K900001 (Data + Cofile) ---> [ QAS ] --- STMS Import ---> [ PRD ]

MODERN Git DEVOPS (gCTS):
[ ABAP ADT ] ---> [ Git Commit / Push ] ---> [ GitHub / GitLab Repo ] ---> [ CI/CD Pipeline ] ---> [ QAS / PRD Deploy ]
```

*   **Klasik Taşıma Sistemi (STMS):** Değişiklikler bir Taşıma İsteğine (Transport Request - TR) kaydedilir. Dışa aktarıldığında işletim sistemindeki `/usr/sap/trans` dizininde `data` ve `cofile` ikili dosyaları üretilir ve hedef sistemlere `tp` aracıyla aktarılır.
*   **gCTS (Git-enabled Change and Transport System):** ABAP Platform 1909+ ile gelen modern DevOps altyapısıdır.
    *   ABAP paketlerini doğrudan Git depolarına (GitHub, GitLab) bağlar.
    *   Yazılımcılar dal (branch) açabilir, çekme istekleri (Pull Request) gönderebilir ve kurumsal CI/CD hatlarıyla kod kalitesi (ABAP Test Cockpit - ATC) kontrollerinden sonra hedef sistemlere otomatik dağıtım yapabilir.

### 10.4. Uygulama Yaşam Döngüsü: Solution Manager'dan Cloud ALM'e Geçiş
*   **SAP Solution Manager (SolMan 7.2):** Kurulumu ve bakımı oldukça maliyetli olan devasa on-premise yönetim platformudur. Standart desteği **2027 yılı sonunda sona erecektir**.
*   **SAP Cloud ALM:** BTP üzerinde çalışan, sıfır bakım maliyetli bulut yerel yönetim platformudur:
    *   *Implementation:* Proje yönetimi, iş süreçleri haritalandırması ve otomatik test yönetimi.
    *   *Operations:* Dağıtık sistemlerin performansını, API entegrasyon hatalarını, arka plan işlerini ve gerçek kullanıcı deneyimini (Real User Monitoring - RUM) anlık olarak izler.

---

## 11. Bilgisayar Bilimleri Perspektifinden Katmanlı Sistem Modeli (Full Stack Architecture)

Tüm SAP ekosisteminin donanımdan son kullanıcı arayüzüne kadar bilgisayar bilimleri soyutlama katmanları:

| Katman | Katman Adı | Temel Teknolojiler ve Protokoller | Bilgisayar Bilimleri Karşılığı |
| :--- | :--- | :--- | :--- |
| **L6** | **Presentation & AI Interface** | SAP Fiori, OpenUI5, SAP Joule, OData v2/v4, WebSockets | Dağıtık MVC Sunum Katmanı, Contextual Conversational AI, UI Component Trees |
| **L5** | **Integration & Network Security** | SAP Cloud Connector, BTP Connectivity, Event Mesh, mTLS, SCIM 2.0, IAS/IPS | Reverse Multiplexed TCP Tunneling, AMQP/MQTT Pub-Sub, Cryptographic Identity Propagation |
| **L4** | **Application & Virtual Machine** | ABAP Platform Kernel (C/C++), RAP Framework, CDS Views, PXA Memory, ENSA2 | Bytecode Virtual Machine, Lock Allocation Engine, Shared Memory Process Multiplexing |
| **L3** | **Data Model & Processing Logic** | Universal Journal (`ACDOCA`, `MATDOC`), Business Partner, SAP LUW Architecture | Denormalized Wide-Column Relational Storage, Asynchronous Two-Phase Commit Processing |
| **L2** | **In-Memory Database Engine** | SAP HANA Indexserver, Column Store, HEX (LLVM), Delta Merge, Vector Engine | SIMD-Accelerated Vector Processing, Dictionary & Bit-Packed Compression, MVCC In-Memory Storage |
| **L1** | **Operating System & Kernel** | SLES/RHEL for SAP, POSIX SHM (`/dev/shm`), XFS Direct I/O, Pacemaker, STONITH | NUMA-Aware Memory Allocation, HugePage Virtual Memory Tuning, Distributed Consensus Clustering |
| **L0** | **Hardware & Microarchitecture** | x86-64 (Intel Xeon / AMD EPYC), IBM Power10 (SMT8, OMI), Terabyte Scale DRAM | Multi-Socket Coherent Bus Architecture, Hardware Hardware Multi-Threading, Direct Bus Attached Storage |