SAP ekosistemi, en alttaki donanım ve Linux çekirdeği optimizasyonlarından başlayıp bellek içi (in-memory) sütun tabanlı depolama motoruna, C++ ile yazılmış sanal makine (ABAP Kernel) katmanına, oradan da dağıtık bulut servisleri ve SaaS ürünlerine uzanan çok katmanlı bir kurumsal bilişim yığıtıdır (enterprise computing stack).

Bu mimariyi bare-metal/işletim sistemi seviyesinden başlayarak en üstteki orkestrasyon katmanına kadar katmanlar halinde modelleyebiliriz.

## 1. Donanım ve İşletim Sistemi Katmanı (Bare-Metal, Virtualization & OS)

SAP sistemlerinin omurgasını oluşturan **SAP HANA**, geleneksel RDBMS'lerin disk merkezli I/O darboğazlarını aşmak için doğrudan donanım mimarisine kenetlenecek şekilde tasarlanmıştır.

```
+---------------------------------------------------------------------------------+
|                                 SAP HANA Runtimes                               |
+---------------------------------------------------------------------------------+
| POSIX Shared Memory (/dev/shm) | NUMA-Aware Threads | Direct Memory Pointers    |
+---------------------------------------------------------------------------------+
| Linux Kernel: SLES for SAP / RHEL for SAP (hugepages, sysctl, transparent HP)   |
+---------------------------------------------------------------------------------+
| Hypervisor (VMware ESXi / KVM) VEYA Bare-Metal (x86-64 Haswell+ / IBM Power)   |
+---------------------------------------------------------------------------------+
| Hardware: NUMA Node 0 (CPUs + RAM) <--- QPI/UPI ---> NUMA Node 1 (CPUs + RAM)   |
+---------------------------------------------------------------------------------+
```

### İşletim Sistemi ve Çekirdek (Kernel) Özelleştirmeleri

- **Desteklenen Dağıtımlar:** SAP HANA, yalnızca kurumsal Linux dağıtımlarında (**SUSE Linux Enterprise Server for SAP Applications - SLES** veya **Red Hat Enterprise Linux for SAP Solutions - RHEL**) çalışır.
- **Bellek Yönetimi ve NUMA (Non-Uniform Memory Access):**
  - Terabaytlarca RAM içeren çok soketli sistemlerde CPU çekirdeklerinin uzak soketteki belleğe erişimi yüksek gecikmeye (latency) yol açar. HANA'nın alt seviye motoru NUMA mimarisini doğrudan algılar (NUMA-aware allocator).
  - İş parçacıkları (threads), veri yapılarının tutulduğu yerel NUMA düğümlerine bağlanır (thread affinity/CPU pinning).
- **Transparent Huge Pages (THP) Paradoksu:** Standart Linux sistemlerinde I/O performansını artıran THP, HANA'da dinamik bellek ayırma sırasında gecikme piklerine (memory fragmentation ve lock contention) neden olduğu için işletim sistemi seviyesinde kapatılır; yerine sayfa tablolarının şişmesini önleyen özel yapılandırmalar tercih edilir.
- **Dosya Sistemleri ve IPC:** Veritabanı katmanında süreçler arası haberleşme için yoğun olarak POSIX shared memory (`/dev/shm`) kullanılır. Veri ve log blokları için `XFS` dosya sistemi standarttır.

### Sanallaştırma ve Donanım Standartları

- **On-Premise Sanallaştırma:** HANA, donanıma doğrudan (bare-metal) kurulabileceği gibi **VMware vSphere (ESXi)** veya **KVM** üzerinde sanal sunucu (VM) olarak da çalıştırılabilir. Buradaki sanal makine, fiziksel CPU soketlerini ve RAM bloklarını doğrudan sanal makineye bağlayan katı kurallara (NUMA-aligned VM sizing) tabidir.
- **Donanım Kurulum Modelleri (Appliance vs. TDI):**
  - *Appliance Modeli:* Donanım üreticisi (HPE, Dell, Lenovo vb.) tarafından işletim sistemi ve HANA önceden yüklenmiş, onaylı anahtar teslim donanım kutusudur.
  - *TDI (Tailored Datacenter Integration):* Kurumun mevcut kurumsal SAN/NAS depolama ve ağ altyapısını kullanmasını sağlayan, SAP donanım sertifikasyon araçlarıyla (`HCMT` / `HWCCT`) onaylanan esnek kurumsal veri merkezi standardıdır.
- **Bulut Altyapısı (IaaS):** AWS (EC2 Nitro / Bare Metal Instances), Microsoft Azure (M-Series / Large Instances) ve GCP üzerinde çalışan HANA sistemleri, temelde hipervizör kısıtlamaları kaldırılmış devasa bellekli hesaplama düğümlerinden (Compute Engine / VM) ibarettir.

## 2. Veritabanı Çekirdeği: SAP HANA Internals ve Bellek Mimarisi

HANA, yalnızca bir SQL motoru değil; hesaplamayı diske veri taşıyarak değil, verinin bellekte durduğu yerde (code pushdown) gerçekleştiren bir paralel hesaplama platformudur. Analiz ve işlem yüklerini aynı bellek bloklarında çalıştıran bir **HTAP (Hybrid Transaction/Analytical Processing)** motorudur. Sütun tabanlı saklama mimarisi, okuma işlemlerini aşırı hızlandırırken yazma işlemlerini zorlaştırır. Bu çelişki **Delta Merge** mekanizması ile aşılır.

```
                           +------------------------+
                           |  SQL Client / OData    |
                           +-----------+------------+
                                       |
+--------------------------------------v------------------------------------------+
| SAP HANA Indexserver (Ana Süreç / Daemon)                                       |
|                                                                                 |
|  +--------------------+  +----------------------+  +-------------------------+  |
|  | SQL Parser / Auth  |  |  Calculation Engine  |  | Plan Optimizer / Codegen|  |
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

- **Sorted Dictionary:** Benzersiz değerlerin alfabetik veya sayısal sıralı dizisidir. Her değere 0'dan başlayan bir tamsayı belirteç (`ValueID`) atanır.
- **Bit-Packed Attribute Vector:** Orijinal satır sırasını koruyan dizidir; ancak değerin kendisini değil, sadece `ValueID`'sini tutar.
  - 3 benzersiz değer için sadece 2 bit (`00`, `01`, `10`) yeterlidir.
  - CPU, `SELECT COUNT(*) WHERE Ulke = 'TR'` sorgusu geldiğinde metin karşılaştırması yapmaz; SIMD komutlarıyla doğrudan `01` bit maskesini vektörel olarak tarar.
- **Sıkıştırma ve Vektörizasyon:** Sütun verileri sözlük kodlaması (Dictionary Encoding) yanı sıra Run-Length Encoding (RLE), Cluster Encoding ve Sparse Encoding ile sıkıştırılır. Sayısal filtrelemeler doğrudan CPU L1/L2 önbelleğinde, SIMD (Single Instruction, Multiple Data) vektör komutları kullanılarak bellekten veri çözülmeden (decompression-free) taranır.

### 2.2. Delta Store ve Delta Merge Algoritması

Sıkıştırılmış ve sıralı ana tabloya (`Main Store`) doğrudan yeni bir satır eklemek, tüm sözlüğün yeniden sıralanmasını ve attribute vector bitlerinin kaydırılmasını gerektirir; bu işlem CPU ve bellek bant genişliğini kilitler. Bu yüzden yazma işlemleri geçici bir tampon olan **Delta Store**'a yazılır.

Zamanla Delta Store büyür ve okuma performansı düşmeye başlar. Bu noktada arka plan birleştirme süreci devreye girer.

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

### 2.3. Delta Merge Tetikleyicileri ve Donanım Riskleri

Delta merge işlemi veritabanı motoru tarafından otomatik olarak yönetilir (`Mergedog` servisi):

- **Tetikleme Kriterleri (Auto Merge Check):**
  - *Delta Boyutu:* Delta belleği belirli bir eşiği (örneğin Main'in %5-10'u veya mutlak MB sınırı) aştığında.
  - *Log Alanı Tüketimi:* Delta günlüğe aşırı yazma yaptığında.
  - *Sorgu Maliyeti:* Delta aramalarının CPU maliyeti belirlenen eşiği aştığında.

- **Geçici Bellek Çiftlenmesi Riski (Memory Footprint Spike):**
  - Adım 2 sırasında, $Main_1$ ve $Delta_1$ hala aktif sorgulara hizmet vermek için RAM'de tutulurken, aynı anda yeni oluşturulan $Main_2$ de belleğe yazılır.
  - Bu durum, birleştirme anında tablonun bellekte **yaklaşık 2 kat yer kaplamasına** ($Size(Main_1) + Size(Delta_1) + Size(Main_2)$) yol açar.
  - Eğer HANA Allocated Memory, işletim sistemi limitlerine yakınsa bu işlem **OOM (Out-of-Memory Killer)** sürecini tetikleyebilir.

- **Optimizasyon Çözümü (Table Partitioning):**
  - Çok büyük tablolar (yüz milyonlarca satır içeren `ACDOCA` veya `MATDOC`), HASH veya RANGE yöntemleriyle fiziksel bölümlere (partitions) ayrılır.
  - Delta merge tüm tabloya değil, yalnızca ilgili küçük bölüme uygulanır. Böylece hem işlem süresi milisaniyeler seviyesine iner hem de birleşme sırasındaki bellek zıplamaları kontrol altında tutulur.

### 2.4. Veri Güvenliği ve Kalıcılık (ACID Garantisi)

RAM uçucu bir bileşendir; bu nedenle HANA iki aşamalı bir disk kalıcılığı (persistence) mimarisi kullanır:

1. **Redo Log Buffer (`/hana/log`):** Bir işlem (transaction) `COMMIT` edildiğinde, sadece yapılan operasyon sıralı olarak diske (WAL mantığı) yazılır. Bu yazma NVMe/SSD üzerinde senkron gerçekleşir.
2. **Savepoints (`/hana/data`):** Genellikle her 5 dakikada bir, bellekteki değişmiş sayfalar (dirty pages) toplu ve asenkron biçimde diske flush edilir. Sistem çökerse en son savepoint diske yüklenir ve ardından gelen loglar replay edilerek tutarlılık sağlanır.

## 3. Uygulama ve Çalışma Zamanı Katmanı: SAP NetWeaver & ABAP Platform

ABAP Platform (tarihsel adıyla NetWeaver Application Server ABAP), çok kullanıcılı ve yüksek eşzamanlılıklı (high-concurrency) kurumsal iş yüklerini yönetmek üzere C ve C++ ile yazılmış hibrit bir sanal makine ve süreç orkestrasyon katmanıdır.

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

### 3.1. Çekirdek Süreçler (Core OS Processes)

- **Internet Communication Manager (`icman`):** Dış dünyadan gelen HTTP(S), RFC-over-HTTP ve WebSocket isteklerini doğrudan karşılayan bağımsız Linux sürecidir. Olay güdümlü (event-driven) I/O çoklaması (`epoll`) kullanarak istemci bağlantılarını kabul eder; gelen yükü **Memory Pipe Interface (MPI)** üzerinden paylaşılan belleğe yazar ve ABAP Dispatcher kuyruğuna sokar.
- **ABAP Dispatcher:** Tüm isteklerin yönlendirme merkezidir. Kuyruktaki işleri iş süreçlerinin (Work Process - WP) türüne ve uygunluğuna göre dağıtır. İstemci ile sunucu arasında TCP oturumunun sürekli tek bir sürece bağlanmasını engeller (stateless bağlantı çoklaması).
- **Work Processes (WP):** C++ ile yazılmış ABAP çekirdeğini (kernel) çalıştıran, işletim sistemi düzeyinde izole süreçlerdir:
  - **DIA (Dialog):** Son kullanıcı ekran isteklerini, REST/OData çağrılarını yürütür. Maksimum çalışma süresi kısıtı (`rdisp/max_wprun_time`) ile korunur.
  - **BTC (Background):** Zaman aşımı kısıtına takılmayan toplu işleri (batch job) yürütür.
  - **UPD / UPD2 (Update):** Veritabanı tutarlılığını sağlamak için iki aşamalı commit (two-phase commit) mantığını çalıştırır. UPD kritik finansal/operasyonel kayıtları (V1), UPD2 ise istatistik veya indeks güncellemeleri gibi kritik olmayan asenkron işleri (V2) tamamlar.
  - **SPO (Spool):** Yazıcı kuyrukları ve PDF/belge dönüştürme işlerini üstlenir.
- **Standalone Enqueue Server (ENSA2):** SAP'nin uygulama seviyesindeki küresel kilit tablosunu (Lock Table) RAM üzerinde tutar. Veritabanındaki satır kilitlerinden (row-level locks) bağımsız olarak, bir iş nesnesinin (örneğin sipariş tablosunun) kullanıcı tarafından değiştirildiği süre boyunca mantıksal olarak kilitlenmesini sağlar. `Enqueue Replicator` süreciyle asenkron yedeklenerek yüksek erişilebilirlik (HA) sunar.
- **SAP Gateway (`gwrd`):** RFC (Remote Function Call) ve CPIC protokollerini yönetir; dış sistemlerden gelen veya dış sistemlere yönlendirilen RFC bağlantılarını (`SM59`), BAPI entegrasyonlarını ve CPI-C haberleşmesini koordine eder.

### 3.2. Bellek Yönetimi ve Durumsuz (Stateless) Bağlantı Modeli

Geleneksel web uygulama sunucularından farklı olarak bir ABAP sistemi, kullanıcının oturum bağlamını (session state) tek bir iş sürecine (process) sabitlemez.

```
Request 1 (HTTP Get) ----> [ DIA Wp0 ] === Attach User Context from EM ===> Process -> Detach EM
Request 2 (HTTP Post) ---> [ DIA Wp3 ] === Attach Same User Context ===> Process -> Detach EM
```

- **SAP Extended Memory (EM):** POSIX paylaşılan bellek segmentleridir (`/dev/shm`). Kullanıcının global değişkenleri, açık iç tabloları (internal tables) ve oturum verileri burada tutulur.
  - İşlem adımı (dialog step) başladığında, Dispatcher isteği boşta olan herhangi bir DIA sürecine verir.
  - İlgili DIA süreci, kullanıcının EM üzerindeki bellek segmentine bellek eşleme (memory mapping / pointer attach) yapar.
  - İşlem bittiğinde bağlam ayrılır (detach); veri EM'de kalır, iş süreci ise bir sonraki farklı kullanıcının isteğine tahsis edilir.
- **Program Execute Area (PXA):** Derlenmiş ABAP bayt kodlarının tutulduğu küresel paylaşılan bellek alanıdır. Kodlar tekrar tekrar diskten okunmaz; PXA üzerinden doğrudan yürütülür.
- **Process Local Heap ve PRIV Mode:** Eğer bir kullanıcının tükettiği bellek yapılandırılan EM sınırını (`em/initial_size_MB`) aşarsa, iş süreci yerel işletim sistemi belleğinden (`malloc`) bellek almaya başlar. Bu duruma **PRIV Mode** (Private Mode) denir.
  - Süreç PRIV moduna girdiğinde, bellek yerel olduğu için başka hiçbir kullanıcıya hizmet veremez ve ilgili oturum sonlanana kadar o kullanıcıya kilitlenir.
  - Sistemde PRIV moduna giren süreç sayısının artması, kuyrukların şişmesine (work process exhaustion) yol açan en kritik darboğazdır.

## 4. ERP Evrimi: ECC'den S/4HANA'ya Geçiş

Klasik SAP ERP (ECC 6.0) ile modern S/4HANA arasındaki temel fark, salt bir UI değişikliği veya veritabanı geçişi değil; veritabanının yeteneklerine göre verinin veri tabanında tutulma şeklinin (data modeling) baştan yazılmasıdır.

|**Kriter**|**SAP ECC 6.0 (Legacy)**|**SAP S/4HANA (Modern)**|
|---|---|---|
|**Veritabanı Desteği**|Veritabanından bağımsız (AnyDB: Oracle, MSSQL, DB2, MaxDB).|Yalnızca SAP HANA (In-Memory).|
|**Finansal Tablo Mimarisi**|İndeks ve toplama (aggregate) tabloları (`BSIS`, `BSAS`, `GLT0`, `BSEG`).|Sadeleştirilmiş tek evrensel finans tablosu (**`ACDOCA`** - Universal Journal).|
|**Lojistik Tablo Mimarisi**|Ayrık stok ve malzeme hareket tabloları (`MKPF`, `MSEG`, `MBEW`, `MARC`).|Sadeleştirilmiş tek evrensel malzeme tablosu (**`MATDOC`**).|
|**İş Ortağı Mimarisi**|Müşteri (`KNA1`) ve Satıcı (`LFA1`) ayrık ana verileri.|Zorunlu **Business Partner (BP)** ve CVI entegrasyonu.|
|**Analitik & İşlem Ayrımı**|OLTP ve OLAP ayrıdır; raporlar için veri SAP BW sistemine kopyalanır.|**HTAP** (Hybrid Transactional/Analytical Processing); aynı tabloda anlık raporlama.|
|**Kullanıcı Arayüzü**|SAP GUI (Özel binary DIAG protokolü).|SAP Fiori (Web tabanlı, UI5, OData üzerinden REST/JSON).|

### Tablo Sadeleştirmesi ve Universal Journal (`ACDOCA` & `MATDOC`)

Geleneksel veritabanlarında `SELECT SUM(Tutar)` çalıştırmak milyonlarca satırda yavaş olduğu için yazılımcılar toplama tabloları tasarlardı. S/4HANA ile HANA'nın bellek içi sütun motoru sayesinde bu tablolar çöpe atılmıştır:

- **Finansal Konsolidasyon (`ACDOCA`):** Genel muhasebe, maliyet muhasebesi (CO), varlık yönetimi (AA) ve kâr merkezi kayıtları tek bir tabloda (`ACDOCA`) birleştirilmiştir.
- **Lojistik Konsolidasyonu (`MATDOC`):** Malzeme hareketleri, stok değerlemeleri ve envanter kayıtları tek bir evrensel tabloda (`MATDOC`) toplanarak kilitlenme (lock contention) süreleri ortadan kaldırılmıştır.
- İndeks tablolarının kaldırılması, veri tabanındaki kilitlenme sürelerini yok etmiş, I/O yükünü onlarca kat düşürmüştür.

## 5. Dağıtım Modelleri: On-Premise, Private Cloud ve Public Cloud

SAP S/4HANA üç temel dağıtım modelinde sunulur. Bu modeller donanım sahipliğinden ziyade **yazılımın genişletilebilirliği (extensibility)** ve **yükseltme (upgrade) kontrolü** sınırlarıyla birbirinden ayrılır.

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

### 1. SAP S/4HANA On-Premise

- Müşteri sunucuları kendi veri merkezinde veya kiraladığı bir IaaS sağlayıcısında (AWS, Azure vb.) barındırır.
- İşletim sistemi (`root`), veritabanı (`SYSTEM/HDB`), SAP Kernel ve ABAP ortamı üzerinde tam kontrol vardır.
- `SE38`, `SE80` gibi işlemlerle standart SAP kaynak kodları değiştirilebilir (modifikasyon), doğrudan sistem tablolarına erişen `Z`'li programlar yazılabilir. Bu durum sonraki versiyon geçişlerini (upgrade) teknik bir kabusa çevirir.

### 2. SAP S/4HANA Cloud, Private Edition (PCE / RISE with SAP)

- Tek kiracılı (single-tenant) bir sistemdir. Altyapı SAP tarafından yönetilen Hyperscaler hesaplarında tutulur.
- On-Premise ile aynı fonksiyonel kod tabanına sahiptir; mevcut eski kodlar buraya taşınabilir.
- İşletim sistemi erişimi kaldırılmıştır; teknik bakım, kernel güncellemeleri ve yedekleme operasyonları SAP'nin SLA sözleşmesi kapsamındadır.

### 3. SAP S/4HANA Cloud, Public Edition

- Gerçek bir **Software-as-a-Service (SaaS)** modelidir. Çok kiracılı (multi-tenant) altyapı üzerinde veya standartlaştırılmış izole ortamlarda barındırılır.
- **Geliştirici Kısıtlamaları:** Klasik ABAP GUI (`SAP GUI`), `SE80`, doğrudan tablo güncellemesi (`INSERT INTO standard_table`) tamamen yasaktır.
- **Zorunlu Güncelleme:** Sistem yılda iki kez SAP tarafından otomatik olarak güncellenir. Müşterinin güncellemeyi reddetme yetkisi yoktur.

## 6. Modern Yazılım Felsefesi: "Clean Core" ve Genişletilebilirlik (Extensibility)

Geçmişte kurumsal firmaların SAP projelerinde yazdığı kontrolsüz modifikasyonlar, ERP sistemlerinin modernize edilmesini engellemiştir. SAP bu sorunu aşmak için modern bilgisayar bilimlerindeki gevşek bağlılık (loose coupling) prensibini getiren **Clean Core** paradigmasını zorunlu kılmaktadır.

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

### Extensibility Seviyeleri

1. **Key User Extensibility:** İş analistlerinin Fiori üzerindeki araçlarla yeni veri alanları eklemesi veya basit doğrulama mantıkları (Validation BAdI) yazmasıdır.
2. **On-Stack Developer Extensibility (ABAP Cloud):** S/4HANA üzerinde kod yazılır ancak sadece `ABAP for Cloud Development` dil sürümü ve **Eclipse ADT (ABAP Development Tools)** kullanılır. Doğrudan standart tablolara SQL atılamaz; yalnızca SAP tarafından açıkça `Released for Key User / Cloud Development` (C1 sözleşmesi) olarak etiketlenmiş CDS View ve API'lar çağrılabilir.
3. **Side-by-Side Extensibility:** Çekirdeğe hiç dokunulmaz. İhtiyaç duyulan uygulama **SAP BTP** üzerinde bağımsız bir mikroservis olarak ayağa kaldırılır; S/4HANA ile yalnızca OData servisleri veya asenkron olaylar (Event Mesh) üzerinden konuşur.

## 7. Bulut Platformu Katmanı: SAP BTP, Ağ ve Kimlik Güvenliği

**SAP BTP**, sistemin PaaS (Platform-as-a-Service) katmanıdır. Farklı SaaS ürünlerinin, kurum içi sistemlerin ve üçüncü parti servislerin buluştuğu entegrasyon merkezidir.

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

### 7.1. SAP Cloud Connector (SCC) Network Internals

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

- **Sıfır Gelen Port (Zero Inbound Rule):** SCC, şirket içi DMZ katmanında çalışan hafif bir Java (JVM) arka plan sürecidir. Şirket güvenlik duvarında internete doğru hiçbir gelen (inbound) porta izin verilmez.
- **Kalıcı Çift Yönlü mTLS Tüneli (Reverse Proxy Flow):**
  1. SCC başlatıldığında, SAP BTP Connectivity Service uç noktasına doğru giden (outbound) bir HTTPS bağlantısı (`Port 443`) başlatır.
  2. Karşılıklı sertifika doğrulaması (**Mutual TLS - mTLS**) yapılarak tünel kurulur.
  3. Tünel bir kez kurulduktan sonra açık kalır; BTP üzerinde çalışan bir mikroservis şirket içindeki bir servisi çağırmak istediğinde, istek bu açık tünel içinden tersine akıtılarak (reverse multiplexing) SCC'ye iletilir.
- **Virtual-to-Internal URL Mapping:** Güvenlik katmanı olarak SCC, iç ağın topolojisini buluttan gizler.
  - *Sanal Adres:* `s4core.virtual.corp:443`
  - *Gerçek Adres:* `10.20.4.112:44300` (HTTPS) veya `sapgw00` (SNC RFC)
  - Yetkisiz URL yolları (`/sap/bc/gui/*` vb.) tünel seviyesinde filtrelenerek sadece onaylanmış servis yollarına (`/sap/opu/odata/sap/API_BUSINESS_PARTNER/*`) erişim izni verilir.

### 7.2. Principal Propagation (Kriptografik Kimlik Aktarımı)

Kullanıcı şifrelerinin buluta aktarılmasını önleyen kriptografik kimlik yayılımıdır:

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

- Kullanıcı BTP'ye giriş yaptığında bir OAuth 2.0 JWT token alır.
- İstek SCC'ye ulaştığında, SCC yerel sertifika yetkilisi (Local CA) aracılığıyla JWT içindeki e-posta/kullanıcı adı adına **kısa ömürlü (short-lived) bir X.509 istemci sertifikası** türetir.
- Bu sertifika S/4HANA'ya sunulur. S/4HANA sertifikadaki Subject adını (ör. `CN=john.doe@company.com`) kendi `USRACEXT` ve `CERTRULE` tablosundaki SAP kullanıcısıyla eşleştirir ve oturum şifresiz, güvenli şekilde açılır.

### 7.3. Kimlik Yaşam Döngüsü: IAS, IPS ve JML Entegrasyonu

Merkezi kimlik yönetimi (IAM), **SAP Cloud Identity Services** bileşenleri olan **IAS (Identity Authentication Service)** ve **IPS (Identity Provisioning Service)** üzerinden yürütülür.

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

#### JML (Joiner - Mover - Leaver) Süreç Akışı

1. **Joiner (İşe Giriş):**
   - İK sistemi (SuccessFactors) yeni çalışan kaydını açar.
   - IPS, SuccessFactors `User/EmpJob` API'sindeki delta değişikliği yakalar.
   - Kullanıcıyı IAS üzerinde oluşturur, BTP alt hesaplarına provision eder ve Cloud Connector üzerinden S/4HANA'da `BAPI_USER_CREATE1` çağırarak ABAP kullanıcısını ve Business Partner (BP) kaydını üretir.
2. **Mover (Pozisyon/Departman Değişimi):**
   - İK sisteminde departman veya iş kodu değiştiğinde IPS, kullanıcının mevcut yetki gruplarını (`Role Collections`) hesaplar.
   - Eski pozisyona ait roller kaldırılır, yenileri otomatik bağlanır (least privilege ilkesi).
3. **Leaver (İşten Ayrılış):**
   - İş akışı sonlandığında İK sistemi kaydı pasife çeker (`status: inactive`).
   - IPS anlık tetikleme ile IAS üzerindeki kullanıcı bayrağını `active: false` yapar.
   - Aktif tüm JWT ve SAML oturumları IAS katmanında geçersiz kılınır; kullanıcı saniyeler içinde tüm SAP bulut ve on-premise ekosisteminden tamamen izole edilir.

## 8. Çevre SaaS Portföyü ve Entegrasyon Matrisi

S/4HANA şirketin dijital çekirdeğidir (core). Özelleşmiş iş kolları ise SAP'nin satın almalar veya bulutta sıfırdan geliştirmeler yoluyla ekosisteme kattığı bağımsız SaaS çözümleri tarafından yürütülür.

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

- **SAP SuccessFactors:** İnsan kaynakları yönetimi (HCM). Çoğunlukla Java/OData üzerinde çalışan, çalışan verilerinin ana kaynağı (master data) olan SaaS ürünüdür. SCIM/IPS ile S/4HANA'daki iş ortağı (Business Partner) kayıtlarıyla eşitlenir.
- **SAP Ariba:** Tedarikçi ağı ve B2B satınalma platformudur. Tedarikçilerle sipariş ve faturalar **cXML** formatında **SAP Integration Suite (Managed Gateway)** üzerinden S/4HANA malzeme yönetimi (MM) modülüyle çift yönlü haberleşir.
- **SAP Concur:** Masraf ve seyahat yönetimi SaaS platformu. Harcama kayıtları REST API'lar üzerinden S/4HANA finansal belgelerine (`ACDOCA`) dönüştürülür.
- **SAP IBP (Integrated Business Planning):** Tedarik zinciri planlama platformu. Arka planda devasa bellek içi algoritmalar ve istatistiksel tahminleme motorları çalıştıran HANA altyapısı üzerinde koşan bulut çözümüdür.

## 9. Bilgisayar Bilimleri Perspektifinden Katman Özeti (Full Stack Model)

SAP mimarisini katmanlı bir bilgisayar bilimi yığıtı olarak okumak, ekosistemin parçalarını anlamlandırmayı kolaylaştırır:

```
+------------------------------------------------------------------------------------+
| KATMAN 5: SUNUM / KULLANICI ARAYÜZÜ (Presentation Layer)                          |
| - SAP Fiori / OpenUI5: MVC Mimarisi, OData v2/v4 Tüketimi, Web Components          |
| - SAP GUI: Binary DIAG Protokolü (Tarihsel Desktop Terminal İstemcisi)             |
+------------------------------------------------------------------------------------+
| KATMAN 4: ENTEGRASYON VE DAĞITIK SİSTEMLER (Distributed Integration & Security)   |
| - BTP Event Mesh (Asenkron AMQP/MQTT Pub-Sub)                                      |
| - SAP Cloud Connector: mTLS Tabanlı TCP/IP Ters Tünelleme (Reverse Proxy)          |
| - Güvenlik: SAML 2.0, OpenID Connect, OAuth 2.0 Bearer JWT Tokens, SCIM 2.0       |
+------------------------------------------------------------------------------------+
| KATMAN 3: İŞ MANTIĞI VE ALAN MODELİ (Domain Logic & Virtual Machine)              |
| - ABAP Kernel VM: Bayt Kodu Yürüten C++ Sanal Makinesi, PXA Shared Memory         |
| - ABAP RESTful Application Programming Model (RAP): CDS Views, Core Engine         |
| - Universal Journal (ACDOCA & MATDOC): Normalize Edilmemiş Geniş Veri Modelleri    |
+------------------------------------------------------------------------------------+
| KATMAN 2: VERİTABANI MOTORU VE PARALEL HESAPLAMA (In-Memory Engine)                |
| - SAP HANA Indexserver: Column Store, Dictionary Encoding, SIMD Vektörizasyonu     |
| - Delta-Main Merge: MVCC Destekli İki Aşamalı Bellek Düzenlemesi                   |
| - Kalıcılık: Write-Ahead Logging (WAL) ve Periyodik Bellek Boşaltma (Savepoints)   |
+------------------------------------------------------------------------------------+
| KATMAN 1: İŞLETİM SİSTEMİ VE DONANIM ÇEKİRDEĞİ (OS & Hardware)                    |
| - Linux Kernel (SLES / RHEL): POSIX Shared Memory (/dev/shm), XFS, Direct I/O     |
| - Donanım Mimarisi: Çok Düğümlü NUMA Optimizasyonu, CPU Affinity, Terabaytlık RAM |
+------------------------------------------------------------------------------------+
```
