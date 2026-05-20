# Sistem Mimarisi - Firebox

Bu doküman, Firebox uygulamasının teknik mimarisini, veri akışını, platform entegrasyonlarını, güvenlik yapılandırmalarını ve kod tasarım desenlerini detaylandırmaktadır.

## 1. Mimari Tasarım İlkeleri

Firebox, **Local-First (Önce Yerel)** ve **Serverless Client (Sunucusuz İstemci)** mimarisine dayanmaktadır. İstemci ile Firebase servisleri arasında hiçbir proxy veya Firebox'a ait bir backend sunucusu bulunmaz. Bu yaklaşım üç kritik avantaj sağlar:
1. **Maksimum Güvenlik**: Kullanıcının gizli Service Account anahtarları veya veritabanı verileri asla üçüncü şahıs sunucularından geçmez.
2. **Sıfır Sunucu Maliyeti**: Uygulama tamamen istemci üzerinde çalıştığı için operasyonel sunucu maliyeti sıfırdır.
3. **Düşük Gecikme (Latency)**: İstekler doğrudan cihazdan Google/Firebase API gateway'lerine iletilir.

```
+---------------------------------------------------------------------------------+
|                                FIREBOX CLIENT                                   |
|                                                                                 |
|  +--------------------+    +-----------------------+    +--------------------+  |
|  |  Presentation UI   | <  |  Riverpod Providers   |  > |  Local DuckDB      |  |
|  |  (Desktop/Mobile)  |    |  (State & Logic)      |    |  (SQL & Analytics) |  |
|  +--------------------+    +-----------------------+    +--------------------+  |
|                                        v                                        |
|                            +-----------------------+                            |
|                            |  Platform Services    |                            |
|                            |  (Service Registry)   |                            |
|                            +-----------------------+                            |
|                                        v                                        |
|                     +--------------------------------------+                    |
|                     |     Google APIs & gRPC Client        |                    |
|                     |     (googleapis / grpc package)      |                    |
|                     +--------------------------------------+                    |
+----------------------------------------|----------------------------------------+
                                         | (Direct HTTPS / gRPC over SSL)
                                         v
                      +--------------------------------------+
                      |      Firebase Cloud Infrastructure   |
                      |  (Firestore / Auth / Cloud Storage)  |
                      +--------------------------------------+
```

## 2. Teknoloji Yığını (Tech Stack)

### Ana Framework ve Dil
- **Framework**: Flutter (Son Güncel Kararlı Sürüm - v3.x.x+)
- **Dil**: Dart (Son Güncel Sürüm - v3.x+)
- **Platformlar**: macOS (bütünleşik), Windows, Linux, iOS, Android.

### Durum Yönetimi (State Management)
- **Paket**: `flutter_riverpod` ve `riverpod_generator`.
- **Neden Riverpod?**:
  - Gelişmiş asenkron veri akış (AsyncValue) desteği.
  - Bağımlılık Enjeksiyonu (Dependency Injection) mekanizmasıyla tam entegrasyon.
  - State'lerin garbage collector tarafından otomatik temizlenmesi (autoDispose).
  - Test edilebilirliğin çok yüksek olması (Provider Container mock'lama kolaylığı).

### Yerel Veritabanı (Local Storage & Analytical Database)
- **Paket**: `dart_duckdb`
- **Neden DuckDB?**:
  - **İlişkisel ve Gelişmiş Analitik Gücü (OLAP)**: DuckDB, hızlı ilişkisel veri depolama sağlamanın yanı sıra, veritabanı içi analitik, veri agregasyonları ve raporlama özellikleri için optimize edilmiş yüksek performanslı bir "Sütun Tabanlı" (Columnar) SQL veritabanıdır.
  - **Firebase Analiz ve Caching**: Firebase NoSQL yapıda olduğundan, karmaşık gruplamalar ve analizler (Örn: "Hangi kullanıcılar geçen ay en çok siparişi verdi?" gibi sorgular) Firestore üzerinde çok maliyetlidir. Firebox, Firestore yedeklerini yerel DuckDB tablolarına senkronize ederek kullanıcıların doğrudan SQL standartlarında karmaşık raporlar ve analizler yapabilmesine imkan tanır.
  - **Yapay Zeka (AI) ve Vektör Desteği**: İleride eklenecek yerel AI asistanları ve RAG (Retrieval-Augmented Generation) özellikleri için DuckDB'nin gelişmiş veri tarama, Parquet/CSV doğrudan okuma ve vektör analitiği yetenekleri kullanılacaktır.
  - **Sıfır Konfigürasyon ve Gömülü Çalışma**: İşletim sistemlerinde yerel bir SQLite gibi çalışır, sunucu kurulumu gerektirmez ve ACID uyumludur.

### Ağ ve Firebase Entegrasyonu
Masaüstü uygulamalarında Firebase Native C++ SDK'leri hantal ve kısıtlı olduğundan, Firebox **Google API REST / gRPC** istemcilerini ve resmi Dart paketlerini kullanır:
- **`googleapis`**: Google Cloud API'leri ile doğrudan REST tabanlı entegrasyon (Firestore, Authentication, Storage API v1).
- **`googleapis_auth`**: Service Account JSON ve Google OAuth 2.0 kimlik doğrulama süreçleri için.
- **`http`**: Özelleştirilmiş HTTP istekleri ve Local Emulator Suite ile haberleşme.
- **`grpc`**: Firestore'un gerçek zamanlı dinleme (Real-time Listening) özellikleri için yüksek performanslı gRPC kanalı.

## 3. Özellik Odaklı Temiz Mimari (Feature-First Clean Architecture)

Firebox kod tabanı, ölçeklenebilirliği artırmak, merge çakışmalarını önlemek ve geliştirme hızını en üst düzeye çıkarmak amacıyla **Feature-First Clean Architecture (Özellik Odaklı Temiz Mimari)** prensiplerine göre tasarlanmıştır.

Tüm kod yapısı öncelikle **kullanıcıya dönük bağımsız modüllere (Features)** bölünmüş, ardından her modül kendi içinde **Data, Domain ve Presentation** katmanlarına ayrılmıştır.

### 3.1. Dizin ve Dosya Yapısı

```
lib/
├── core/                  # Uygulama genelinde paylaşılan altyapı kodları
│   ├── database/          # DuckDB servisleri, veritabanı ilklendirme, SQL şemaları ve migration script'leri
│   ├── security/          # Güvenli depolama (flutter_secure_storage) servisleri (Keychain/Keystore)
│   ├── theme/             # Tasarım sistemi, global Dark-first teması ve renk paletleri
│   ├── utils/             # Evrensel yardımcılar (tarih formatlama, veri tipi dönüştürücüler)
│   ├── constants/         # Evrensel sabitler, hata kodları ve API limitleri
│   └── widgets/           # Global paylaşılan widget'lar (Komut Paleti, özel diyaloglar, animasyonlu yükleyiciler)
│
├── features/              # Bağımsız kullanıcı özellikleri (Feature-First)
│   │
│   ├── workspace/         # Workspace ve Bağlantı Yönetimi Özelliği
│   │   ├── data/          # Google OAuth, Service Account, CLI, Emulator veri kaynakları ve Workspace DuckDB tabloları
│   │   ├── domain/        # Workspace iş modelleri, doğrulama kuralları ve repository kontratları
│   │   └── presentation/  # Workspace listeleri, bağlantı ekranları ve Riverpod durum sağlayıcıları
│   │
│   ├── firestore/         # Firestore Gezgini ve Sorgu Motoru Özelliği
│   │   ├── data/          # Firestore REST/gRPC veri kaynakları, döküman DTO'ları ve DuckDB senkronizasyon cache tabloları
│   │   ├── domain/        # Döküman ve koleksiyon modelleri, sorgu oluşturucu kuralları
│   │   └── presentation/  # Tablo/Ağaç/JSON Görünümleri, Split View ve sorgulama paneli UI'ı
│   │
│   ├── auth/              # Firebase Authentication Yönetici Paneli Özelliği
│   │   ├── data/          # Auth Admin API sarmalayıcıları, CSV/JSON toplu aktarım araçları
│   │   ├── domain/        # Auth kullanıcı nesneleri, custom claims veri yapıları
│   │   └── presentation/  # Kullanıcı tabloları, Custom Claims JSON editörü ve toplu işlem UI'ı
│   │
│   ├── storage/           # Firebase Storage Dosya ve Klasör Yöneticisi Özelliği
│   │   ├── data/          # Cloud Storage API servisleri, yükleme/indirme kuyruk tabloları (DuckDB)
│   │   ├── domain/        # Dosya/klasör düğümleri (nodes), aktif transfer kuyruk modelleri
│   │   └── presentation/  # Dosya gezgini, sürükle-bırak paneli, medya önizleyicileri ve kuyruk yöneticisi UI'ı
│   │
│   ├── scripting/         # JS Scripting Shell Özelliği
│   │   ├── data/          # QuickJS VM entegrasyonu, JS console loglarının DuckDB veritabanında loglanması
│   │   ├── domain/        # JS VM çalışma kuralları, scripting oturum modelleri
│   │   └── presentation/  # Renklendirmeli kod editörü, canlı terminal çıktısı UI'ı
│   │
│   └── analytics/         # Yerel DuckDB Analitik & Raporlama Modülü (Yeni Özellik)
│       ├── data/          # Firestore yerel yedek senkronizatörleri, SQL rapor oluşturma servisleri
│       ├── domain/        # Analitik gruplama mantığı, veri görselleştirme modelleri, AI prompt kuralları
│       └── presentation/  # SQL Çalıştırma Alanı (Console), analitik grafikler ve AI raporlama paneli UI'ı
│
└── main.dart              # Uygulama giriş noktası ve DuckDB ilklendirme lojiği
```

### 3.2. Katmanların Rolleri (Feature-Level Layers)

Her bir feature klasörünün altındaki katmanlar şu sorumlulukları üstlenir:

1. **Data Katmanı**:
   - **Datasources (Veri Kaynakları)**: Doğrudan API çağrılarını (`http`, `grpc`, `googleapis`) veya yerel veritabanı (DuckDB SQL) işlemlerini gerçekleştirir.
   - **Models / DTOs**: JSON verilerini Dart nesnelerine dönüştürür. API'lerden dönen ham verilerin temizlenmesi ve formatlanması burada yapılır.
   - **Repositories (Implementations)**: Domain katmanında tanımlanan arayüzleri (interfaces) doldurarak veriyi çeker, önbelleğe (DuckDB) yazar ve domain katmanına saf veri sunar.

2. **Domain Katmanı**:
   - **Entities (Varlıklar)**: UI katmanında doğrudan kullanılacak, tamamen saf Dart nesneleridir. Herhangi bir harici kütüphaneye veya API modeline bağımlı değillerdir.
   - **Repositories (Contracts)**: Veri çekme operasyonlarının imzasını tanımlayan soyut sınıflardır. Data katmanı bu arayüzleri uygulamak zorundadır.
   - **Use Cases / Logic (İsteğe Bağlı)**: Karmaşık iş kurallarını (Örn: "Bir döküman taşınırken aynı anda validasyon kurallarını kontrol etme") sarmalayan sınıflardır.

3. **Presentation Katmanı**:
   - **UI Widgets**: Flutter arayüz bileşenleri, sayfalar, responsive paneller ve diyaloglar bu katmanda yer alır.
   - **State Providers (Riverpod)**: UI'ın dinlediği durumları (state) yönetir. Kullanıcı etkileşimlerini alarak repository'ler aracılığıyla veri işlemlerini tetikler ve sonucu UI'a asenkron (`AsyncValue`) olarak aktarır.

## 4. Güvenlik ve Kimlik Doğrulama Mimarisi (Security & Auth)

Firebox, hassas sistem yöneticisi (Admin) yetkileriyle çalıştığından, güvenlik en üst önceliktir.

### 4.1. Hassas Verilerin Depolanması (Secure Storage)
Service Account özel anahtarları (Private Keys) ve OAuth yenileme token'ları (Refresh Tokens) asla düz metin olarak yerel veritabanında saklanmaz.
- **Mekanizma**: `flutter_secure_storage` paketi kullanılır.
- **Platform Entegrasyonları**:
  - **macOS**: Apple Keychain Services.
  - **Windows**: Data Protection API (DPAPI) ile şifrelenmiş Credential Manager.
  - **Linux**: libsecret (Gnome Keyring / KWallet).
  - **iOS**: Keychain.
  - **Android**: Android Keystore (AES ile şifrelenmiş SharedPreferences).

### 4.2. Kimlik Doğrulama Yöntemleri ve Token Döngüsü
Uygulama üç farklı kimlik doğrulama yöntemi sunar:
1. **Service Account JSON**: Kullanıcı yüklediğinde, dosya çözümlenir ve `googleapis_auth` kullanılarak geçici (1 saatlik geçerlilik süresi olan) bir JWT Access Token alınır. Bu token hafızada (RAM) tutulur ve süresi bittiğinde özel anahtar ile yerelde arka planda otomatik olarak yenilenir.
2. **Google OAuth 2.0**: Masaüstünde yerel bir HTTP sunucu (`localhost:port`) açılarak tarayıcıya yönlendirme yapılır. Google'dan gelen yetkilendirme kodu (auth code) alınarak access/refresh token çifti oluşturulur.
3. **Firebase CLI Entegrasyonu**: Uygulama açılışında masaüstündeki Firebase CLI konfigürasyon dosyalarını tarar (`~/.config/configstore/firebase-tools.json`). Eğer geçerli bir oturum varsa, buradaki kimlik bilgilerini içe aktararak kullanıcının projelerini anında listeler.

## 5. JavaScript Sanal Makinesi ve Scripting Altyapısı (Scripting Engine)

Firebox'ın en ayırt edici özelliklerinden biri, kullanıcının Firestore üzerinde toplu işlemler yapmasını sağlayan entegre JavaScript kabuğudur (Shell).

### 5.1. Çalışma Mantığı (Scripting Bridge)
JS Scripting modülü, **`flutter_js`** paketi kullanılarak izole edilmiş bir **QuickJS/V8** sanal makinesinde çalışır.

```
+------------------------------------+       Dart-JS Bridge       +------------------------------------+
|         JavaScript Runtime         | -------------------------> |            Dart Runtime            |
|                                    | <------------------------- |                                    |
|  const db = firebox.firestore();   |    JSON RPC / Messaging    |  // Dart Firestore Service         |
|  await db.doc('users/1').update({  |                            |  Future<void> updateDocument(...)  |
|    role: 'admin'                   |                            |  {                                 |
|  });                               |                            |    // REST API çağrısı yapar       |
|                                    |                            |  }                                 |
+------------------------------------+                            +------------------------------------+
```

### 5.2. JS Nesne Köprüsü Tasarımı
Sanal makine başlatıldığında, global kapsamda (global scope) aşağıdaki nesneler enjekte edilir:
- **`console.log(...args)`**: Çıktıları yakalayıp Dart tarafındaki konsol widget'ına yönlendirir.
- **`firebox`**: Dart tarafına mesaj gönderen özel köprü:
  ```javascript
  const firebox = {\n    firestore: () => ({\n      collection: (path) => ({\n        get: () => sendMessageToDart('firestore_get', { path }),\n        add: (data) => sendMessageToDart('firestore_add', { path, data })\n      }),\n      doc: (path) => ({\n        get: () => sendMessageToDart('firestore_doc_get', { path }),\n        set: (data) => sendMessageToDart('firestore_doc_set', { path, data }),\n        update: (data) => sendMessageToDart('firestore_doc_update', { path, data }),\n        delete: () => sendMessageToDart('firestore_doc_delete', { path })\n      })\n    }),\n    auth: () => ({\n      listUsers: (maxResults) => sendMessageToDart('auth_list_users', { maxResults }),\n      updateUser: (uid, data) => sendMessageToDart('auth_update_user', { uid, data })\n    })\n  };
  ```
- **Hata Yönetimi (Error Handling)**: JS tarafında fırlatılan istisnalar (exceptions) yakalanarak Dart tarafında satır numarası ve hata mesajıyla birlikte konsolda kırmızı renkle gösterilir.

## 6. Arka Plan İşlem Yönetimi (Background Work & Multi-Threading)

Masaüstü ve mobil cihazlarda büyük veri içe/dışa aktarımları (Data Migration) ve büyük dosya yükleme/indirme işlemleri kullanıcı arayüzünü (UI Thread) dondurmamalıdır.

### 6.1. Dart Isolates
Büyük JSON/CSV dosyalarının işlenmesi, parsing işlemleri ve eş zamanlı ağ isteklerinin yönetimi için Dart'ın **`Isolate`** yapısı kullanılır:
- **`compute`** fonksiyonu kullanılarak işlem yoğunluğu olan (CPU-bound) CSV'den JSON'a dönüştürme ve veri doğrulama işlemleri arka plan isolate'ine aktarılır.
- **Isolate Port Haberleşmesi**: İlerleme durumu (progress percentage) ve anlık hız bilgileri ana isolate'e (UI Thread) gönderilerek ilerleme çubukları gerçek zamanlı güncellenir.

### 6.2. Asenkron Görev Havuzu (Task Queue Manager)
Cloud Storage yükleme/indirme işlemleri için aynı anda en fazla 3 dosyanın transfer edilmesine izin veren, diğerlerini sırada (pending) tutan bir asenkron kuyruk yönetimi (`Concurrency Controller`) uygulanır.
