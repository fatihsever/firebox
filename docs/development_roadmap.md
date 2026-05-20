# Geliştirme Yol Haritası - Firebox

Bu doküman, Firebox projesinin teknoloji seçimlerini, paket bağımlılıklarını, Feature-First Clean Architecture yapısını, 6 ana faza yayılmış geliştirme takvimini, test süreçlerini ve platformlara göre canlıya alma (dağıtım) stratejilerini içerir.

---

## 1. Paket Bağımlılıkları (Dependencies)

Flutter projesinde kullanılacak ana paketlerin ve kütüphanelerin listesi ve kullanım amaçları:

```yaml
dependencies:
  flutter:
    sdk: flutter

  # State Management & Dependency Injection
  flutter_riverpod: ^2.5.1
  riverpod_generator: ^2.4.0

  # Local Storage & Analytical Database (DuckDB)
  dart_duckdb: ^0.1.0            # Yüksek performanslı gömülü ilişkisel analitik veritabanı
  path_provider: ^2.1.3           # Veritabanı dosyalarının platformlara özel güvenli yerlerini belirleme
  path: ^1.9.0                    # Dosya yolu yönetimi

  # Security & Credentials
  flutter_secure_storage: ^9.2.2  # Service Account private key'lerini ve Lisans JWT'lerini OS güvenli kasasında saklama

  # Network & Protocols
  googleapis: ^13.1.0             # REST API tabanlı Firebase ve GCP çağrıları
  googleapis_auth: ^1.6.0         # Google OAuth 2.0 ve Service Account JWT token üretimi
  http: ^1.2.1                    # Özel API istekleri ve Local Emulator Suite desteği
  grpc: ^3.2.4                    # Firestore Real-time dinlemeler için yüksek hızlı kanal

  # Payments & Monetization (YENİ)
  adapty_flutter: ^3.15.5      # Adapty - Mobil (iOS & Android) IAP/Abonelik Entegrasyonu
  url_launcher: ^6.3.0            # Stripe / Paddle ödeme linklerini harici tarayıcıda açmak için

  # UI/UX & Responsive Utilities
  window_manager: ^0.3.8          # Masaüstü pencere boyutlandırıcı, sürükleme ve multi-window yönetimi
  file_picker: ^8.0.3             # Dosya ve klasör seçme araçları
  font_awesome_flutter: ^10.7.0    # Gelişmiş modern ikon kütüphanesi
  flutter_spinkit: ^5.2.1          # Şık ve modern yükleme animasyonları
  fl_chart: ^0.66.0               # Analitik modül için şık pasta, çizgi ve bar grafikleri

  # Maps & Location Visualization
  flutter_map: ^6.1.0             # Geopoint koordinatları için OpenStreetMap harita altlığı
  latlong2: ^0.9.1                # Coğrafi harita hesaplamaları ve marker konumlandırma

  # Media Player & Text Editor (Storage Module)
  video_player: ^2.8.6            # Storage video dosyalarını uygulama içinden oynatma
  audioplayers: ^6.0.0            # Storage ses dosyalarını önizleme
  code_text_field: ^2.3.0         # JS scripting, SQL konsolu ve JSON için renklendirmeli kod editörü

  # Scripting Engine
  flutter_js: ^0.8.1              # JavaScript kodlarını çalıştırmak için izole QuickJS sanal makinesi

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  # Code Generation
  build_runner: ^2.4.9
  riverpod_analyzer_utils: ^0.1.2
```

> **DuckDB Dinamik Kütüphane Yönetimi**: `dart_duckdb` paketi, işletim sistemlerinin yerel dinamik kütüphanelerine (`libduckdb.dylib`, `duckdb.dll`, `libduckdb.so`) dinamik olarak bağlanır. Masaüstü platformlarında (macOS, Windows, Linux) bu yerel kütüphaneler uygulamanın `assets/` dizini altına eklenerek derleme aşamasında platformun yerel kütüphane yollarına kopyalanır. Mobil platformlarda (Android, iOS) ise paket içinde gömülü gelen C/C++ DuckDB kütüphaneleri otomatik olarak yüklenir.

---

## 2. Geliştirme Fazları ve Süreç Takvimi (Sprints)

Proje, toplam **18 haftada** tamamlanacak şekilde 6 ana faza bölünmüştür. Kod tabanı **Feature-First Clean Architecture** prensiplerine göre yapılandırılacaktır.

```
+----------------------------------------------------------------------------------+
| F1: Altyapı & Lisans (H1-3) -> F2: Firestore (H4-7) -> F3: Auth (H8-10)          |
|                                                                                  |
| F4: Storage (H11-13) -> F5: Scripting (H14-16) -> F6: Test & Release (H17-18)    |
+----------------------------------------------------------------------------------+
```

### FAZ 1: Altyapı, Yerel Depolama, Lisans Doğrulama ve Workspace Yönetimi (Hafta 1 - 3)
- **Hafta 1**: Flutter projesinin kurulması. **Feature-First Clean Architecture** klasör hiyerarşisinin (`lib/core/` ve `lib/features/workspace/`) oluşturulması. Global tasarım sistemi, renk paletleri ve dark-theme altyapısının kodlanması.
- **Hafta 2**: **DuckDB** veritabanı entegrasyonunun yapılması. SQL tablo şemalarının (`workspaces`, `saved_queries`, `transfer_history`, `app_settings`, `licensing_info`) oluşturulması. Dart `DuckDbService` ve `LicensingService` bağlantı katmanlarının yazılması. `flutter_secure_storage` entegrasyonu.
- **Hafta 3**: `features/workspace/` modülünün data (Google OAuth, Service Account, CLI, Emulator veri kaynakları), domain (Workspace entities ve repository tanımları) ve presentation (Workspace yönetim ekranları, Riverpod state notifier'ları) katmanlarının tamamlanması. Lisans kontrolünün (JWT ve çevrimdışı 7 günlük esneklik) entegre edilmesi.

### FAZ 2: Firestore Explorer ve Sorgu Motoru (Hafta 4 - 7)
- **Hafta 4**: `features/firestore/` modülünün iskeletinin kurulması. Koleksiyon ve döküman gezinme ağacının (Sidebar & Navigation Tree) gRPC canlı dinleme (realtime listening) servisleriyle birlikte data/domain katmanlarında yazılması.
- **Hafta 5**: Tablo Görünümü (Table View) ve Ağaç Görünümü (Tree View) presentation bileşenlerinin geliştirilmesi. Hücrelere çift tıklayarak satır içi (inline) düzenleme yeteneklerinin eklenmesi. Pro özellikleri kilitleyen yumuşak ödeme duvarlarının (soft paywall) tasarlanması.
- **Hafta 6**: Entegre JSON Metin Editörünün yapılması ve veri tipi doğrulama kurallarının yazılması. Gelişmiş Sorgu Oluşturucu (Query Builder) arayüzünün yapılması ve sorguların yerel DuckDB'ye (`saved_queries` tablosu) kaydedilmesi.
- **Hafta 7**: Geopoint harita entegrasyonu ve Firestore resim önizleme bileşenlerinin yazılması. Döküman kopyalama, taşıma ve işlem bütünlüğünü koruyan yeniden adlandırma (Rename Transaction) lojiğinin tamamlanması.

### FAZ 3: Firebase Authentication Yönetici Paneli (Hafta 8 - 10)
- **Hafta 8**: `features/auth/` modülünün kurulması. Auth kullanıcı listesi, e-posta/UID arama, filtreleme ve tarih bazlı sıralama veri kaynaklarının (Data katmanı) ve UI ekranlarının (Presentation katmanı) yapılması.
- **Hafta 9**: Kullanıcı detay paneli, şifre sıfırlama, e-posta değiştirme, hesap dondurma ve silme aksiyonlarının repository servisleriyle entegre edilmesi.
- **Hafta 10**: Custom Claims JSON editörünün geliştirilmesi. CSV/JSON formatında Auth kullanıcılarını şifre hash'leriyle birlikte içe ve dışa aktarma (Import/Export) süreçlerinin yazılması.

### FAZ 4: Firebase Storage Entegrasyonu ve Medya Oynatıcılar (Hafta 11 - 13)
- **Hafta 11**: `features/storage/` modülünün kurulması. Storage klasör gezgini, dosya listeleme, silme ve arama arayüzlerinin oluşturulması. Klasör simülasyon mekanizmasının (.placeholder) kodlanması.
- **Hafta 12**: Sürükle-bırak (Drag and Drop) desteğinin eklenmesi. İndirme ve Yükleme Kuyruğu (Queue Manager) tasarımı; eşzamanlı transfer kontrolörü, duraklat/devam et lojiğinin DuckDB transfer kuyruk tablolarıyla entegrasyonu. (Premium limit kontrollerinin eklenmesi).
- **Hafta 13**: Entegre video (MP4/WebM), ses (MP3/WAV) ve resim oynatıcı/önizleyici panellerinin yazılması. Dosya metadata (MIME, Cache-Control, Custom) düzenleyicisinin tamamlanması.

### FAZ 5: JS Scripting Shell ve DuckDB Analitik Modülü (Hafta 14 - 16)
- **Hafta 14**: `features/scripting/` modülünün kurulması. `flutter_js` sanal makinesinin entegrasyonu. Dart tarafındaki Firestore/Auth API servislerini JS VM'e bind eden köprünün (Dart-JS Bridge) yazılması.
- **Hafta 15**: Renklendirmeli script editörünün ve canlı konsol çıktı terminalinin (Console Log Viewer) yapılması. Sık kullanılan şablonların eklenmesi.
- **Hafta 16**: `features/analytics/` (Yerel DuckDB Analitik & Raporlama) modülünün geliştirilmesi. Firestore yerel yedek senkronizasyon mekanizmasının yazılması. SQL konsol sandbox arayüzünün yapılması, veri görselleştirme grafiklerinin (`fl_chart`) entegre edilmesi ve AI asistan desteğinin (analiz yorumlayıcı) eklenmesi.

### FAZ 6: Test, Ödeme Entegrasyonları ve Platform Sürümleri (Hafta 17 - 18)
- **Hafta 17**: Tüm modüllerin entegrasyon testleri. **Adapty**, **Stripe Checkout** ve kurumsal toplu lisanslama doğrulama testlerinin yapılması. Performans optimizasyonları (Büyük veri setleriyle çalışırken DuckDB indeksleme stratejileri). Bellek kaçaklarının (Memory Leak) tespiti ve giderilmesi.
- **Hafta 18**: İşletim sistemlerine özel (macOS, Windows, Linux, iOS, Android) paketleme, kod imzalama (signing) ve mağaza süreçlerinin tamamlanması.

---

## 3. Test Stratejisi

Güvenli ve hatasız bir çalışma için 4 aşamalı test planı uygulanır:

1. **Birim Testleri (Unit Tests - Dart & Riverpod)**:
   - Tüm veri modellerinin (DTO) JSON serileştirme testleri.
   - Riverpod state notifier sınıflarının iş mantığı testleri.
   - Güvenli depolama (`flutter_secure_storage`) okuma-yazma testleri.
   - JWT imzalarının geçerlilik süresi ve yetki sınırları testleri.
2. **Ödeme ve Satın Alma Entegrasyon Testleri (Payment Sandboxing - YENİ)**:
   - **Adapty Sandbox**: iOS TestFlight ve Android Google Play Licensing test hesaplarıyla uygulama içi satın alma senaryolarının (Satın alma, yenileme, iptal etme, geri yükleme) test edilmesi.
   - **Stripe / Paddle Test Mode**: Masaüstünde kredi kartı satın alma akışının tamamlanması ve sunucu webhooks'larının başarıyla tetiklenip veritabanındaki lisansı aktif hale getirdiğinin doğrulanması.
   - **Çevrimdışı Mod Sınır Testi**: Cihaz zamanını ileri alarak 7 günlük çevrimdışı sürenin aşılması durumunda premium özelliklerin başarıyla kapanıp kapanmadığının kontrolü.
3. **Entegrasyon Testleri (Integration Tests - Firebase Emulator & DuckDB)**:
   - Yerel Firebase Emulator Suite'e bağlanarak Firestore döküman oluşturma/güncelleme/silme senaryoları.
   - DuckDB ilişkisel bütünlük, veri yazma, asenkron sorgular ve agregasyon (GROUP BY) testleri.
   - JS Scripting VM'in emülatör üzerinde başarıyla veri okuyup yazabildiğinin doğrulanması.
   - Storage dosya yükleme ve indirme kuyruğunun veri bütünlüğü testleri.
4. **Kullanıcı Kabul Testleri (UAT)**:
   - Büyük veri setleri (100,000+ döküman) ile DuckDB yerel senkronizasyonunun ve tablo görünümünün performans ve bellek testleri.
   - İnternet kesintisi senaryolarında uygulamanın hata yakalama ve otomatik yeniden bağlanma davranışlarının ölçülmesi.

---

## 4. Dağıtım ve Dağıtım Süreçleri (CI/CD)

Firebox, her işletim sistemine özgü yerel paket biçimlerinde dağıtılacaktır.

### CI/CD Boru Hattı (GitHub Actions)
- Her `main` branch push işleminde ve yeni sürüm etiketinde (tag) otomatik derleme süreçleri tetiklenir.
- Kod kalitesi için `flutter analyze` ve testler için `flutter test` adımları zorunludur.

### Masaüstü Sürümleri Dağıtımı
- **Windows**:
  - `msix` paketi oluşturulur.
  - Windows SDK aracılığıyla geliştirici sertifikasıyla imzalanır.
  - Windows Store veya doğrudan `.msix` kurulum dosyası olarak sunulur.
- **macOS**:
  - `App Store` dağıtımı ve harici dağıtım için Xcode üzerinden arşivlenir.
  - Apple Developer Certificate ile kod imzalanır.
  - Apple Notarization (Noter onaylama) sürecinden geçirilerek `.dmg` veya `.app` uzantısıyla paketlenir.
- **Linux**:
  - `Snapcraft` ile Snap paketi veya taşınabilir `AppImage` formatında derlenir.

### Mobil Sürümleri Dağıtımı
- **Android**: Google Play Console üzerinden dağıtılmak üzere `App Bundle (AAB)` formatında derlenir ve Play Store'da yayınlanır.
- **iOS**: Apple Developer hesabı üzerinden TestFlight'a yüklenir, ardından App Store'da yayınlanmak üzere `.ipa` formatında paketlenir.
