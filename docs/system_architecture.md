# Sistem Mimarisi - Firebox

Bu doküman, Firebox uygulamasının teknik mimarisini, veri akışını, platform entegrasyonlarını, güvenlik yapılandırmalarını ve kod tasarım desenlerini detaylandırmaktadır.

---

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
|  |  Presentation UI   | <  |  Riverpod Providers   |  > |  Local Isar DB     |  |
|  |  (Desktop/Mobile)  |    |  (State & Logic)      |    |  (Workspaces/Settings) |
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

---

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

### Yerel Veritabanı (Local Storage)
- **Paket**: `isar` (veya alternatif olarak yüksek performanslı ilişkisel ihtiyaçlar için `sqlite3`).
- **Neden Isar?**:
  - Flutter için özel yazılmış ACID uyumlu, son derece hızlı NoSQL veritabanı.
  - Tip güvenliği (Type-safety) ve Dart sınıflarından doğrudan şema oluşturma.
  - Reactive sorgular (veritabanı değiştiğinde UI'ı otomatik tetikleme).
  - Masaüstü ve mobil platformlarda sıfır konfigürasyonla çalışabilme.

### Ağ ve Firebase Entegrasyonu
Masaüstü uygulamalarında Firebase Native C++ SDK'leri hantal ve kısıtlı olduğundan, Firebox **Google API REST / gRPC** istemcilerini ve resmi Dart paketlerini kullanır:
- **`googleapis`**: Google Cloud API'leri ile doğrudan REST tabanlı entegrasyon (Firestore, Authentication, Storage API v1).
- **`googleapis_auth`**: Service Account JSON ve Google OAuth 2.0 kimlik doğrulama süreçleri için.
- **`http`**: Özelleştirilmiş HTTP istekleri ve Local Emulator Suite ile haberleşme.
- **`grpc`**: Firestore'un gerçek zamanlı dinleme (Real-time Listening) özellikleri için yüksek performanslı gRPC kanalı.

---

## 3. Katmanlı Yazılım Mimarisi (Layered Architecture)

Firebox kod tabanı, **Temiz Mimari (Clean Architecture)** ilkelerine göre 4 ana katmana ayrılmıştır:

```
lib/
├── core/                  # Evrensel yardımcı araçlar, sabitler ve ağ yapılandırmaları
│   ├── theme/             # Uygulama temaları ve renk paletleri
│   ├── utils/             # Ortak yardımcı fonksiyonlar (tarih formatlama, dosya dönüştürücüler)
│   └── constants/         # API uç noktaları, yerel DB isimleri, hata kodları
│
├── data/                  # Veri erişim katmanı (Data Access Layer)
│   ├── datasources/       # Yerel (Isar) ve Uzak (Firebase REST) veri kaynakları
│   ├── models/            # JSON serileştirme ve DTO (Data Transfer Object) tanımları
│   └── repositories/      # UI/Domain katmanına temiz veri sunan repolar (Repository pattern)
│
├── domain/                # İş kuralları katmanı (İsteğe bağlı - büyük modüller için)
│   └── entities/          # Saf Dart iş modelleri (UI'da gösterilecek temiz nesneler)
│
└── presentation/          # Sunum katmanı (UI & State)
    ├── common/            # Ortak widget'lar (Özel butonlar, input'lar, kartlar)
    ├── workspace/         # Workspace yönetim ekranları ve state yönetim kodları
    ├── firestore/         # Firestore gezgini, sorgu ekranları, tablo/ağaç/JSON widget'ları
    ├── auth/              # Auth admin ekranları, custom claims editörü, import/export UI
    ├── storage/           # Storage dosya tarayıcı, upload/download kuyruk paneli
    └── scripting/         # JS Shell editörü, konsol ekranları
```

---

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

---

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
  const firebox = {
    firestore: () => ({
      collection: (path) => ({
        get: () => sendMessageToDart('firestore_get', { path }),
        add: (data) => sendMessageToDart('firestore_add', { path, data })
      }),
      doc: (path) => ({
        get: () => sendMessageToDart('firestore_doc_get', { path }),
        set: (data) => sendMessageToDart('firestore_doc_set', { path, data }),
        update: (data) => sendMessageToDart('firestore_doc_update', { path, data }),
        delete: () => sendMessageToDart('firestore_doc_delete', { path })
      })
    }),
    auth: () => ({
      listUsers: (maxResults) => sendMessageToDart('auth_list_users', { maxResults }),
      updateUser: (uid, data) => sendMessageToDart('auth_update_user', { uid, data })
    })
  };
  ```
- **Hata Yönetimi (Error Handling)**: JS tarafında fırlatılan istisnalar (exceptions) yakalanarak Dart tarafında satır numarası ve hata mesajıyla birlikte konsolda kırmızı renkle gösterilir.

---

## 6. Arka Plan İşlem Yönetimi (Background Work & Multi-Threading)

Masaüstü ve mobil cihazlarda büyük veri içe/dışa aktarımları (Data Migration) ve büyük dosya yükleme/indirme işlemleri kullanıcı arayüzünü (UI Thread) dondurmamalıdır.

### 6.1. Dart Isolates
Büyük JSON/CSV dosyalarının işlenmesi, parsing işlemleri ve eş zamanlı ağ isteklerinin yönetimi için Dart'ın **`Isolate`** yapısı kullanılır:
- **`compute`** fonksiyonu kullanılarak işlem yoğunluğu olan (CPU-bound) CSV'den JSON'a dönüştürme ve veri doğrulama işlemleri arka plan isolate'ine aktarılır.
- **Isolate Port Haberleşmesi**: İlerleme durumu (progress percentage) ve anlık hız bilgileri ana isolate'e (UI Thread) gönderilerek ilerleme çubukları gerçek zamanlı güncellenir.

### 6.2. Asenkron Görev Havuzu (Task Queue Manager)
Cloud Storage yükleme/indirme işlemleri için aynı anda en fazla 3 dosyanın transfer edilmesine izin veren, diğerlerini sırada (pending) tutan bir asenkron kuyruk yönetimi (`Concurrency Controller`) uygulanır.
