# Geliştirme Yol Haritası - Firebox

Bu doküman, Firebox projesinin teknoloji seçimlerini, paket bağımlılıklarını, 6 ana faza yayılmış geliştirme takvimini, test süreçlerini ve platformlara göre canlıya alma (dağıtım) stratejilerini içerir.

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

  # Local Storage & Database
  isar: ^3.1.0+3
  path_provider: ^2.1.3

  # Security & Credentials
  flutter_secure_storage: ^9.2.2

  # Network & Protocols
  googleapis: ^13.1.0
  googleapis_auth: ^1.6.0
  http: ^1.2.1
  grpc: ^3.2.4

  # UI/UX & Responsive Utilities
  window_manager: ^0.3.8          # Masaüstü pencere kontrolü
  file_picker: ^8.0.3             # Dosya seçme ve indirme konumlandırma
  font_awesome_flutter: ^10.7.0    # Gelişmiş ikon seti
  flutter_spinkit: ^5.2.1          # Şık yükleme animasyonları

  # Maps & Location Visualization
  flutter_map: ^6.1.0             # Geopoint harita entegrasyonu
  latlong2: ^0.9.1                # Coğrafi hesaplamalar

  # Media Player & Text Editor (Storage Module)
  video_player: ^2.8.6            # Storage video önizleme
  audioplayers: ^6.0.0            # Storage ses dosyaları oynatma
  code_text_field: ^2.3.0         # JS ve JSON için renklendirmeli editör

  # Scripting Engine
  flutter_js: ^0.8.1              # JS betikleri için sanal makine (QuickJS)

dev_dependencies:
  flutter_test:
    sdk: flutter
  
  # Code Generation
  build_runner: ^2.4.9
  isar_generator: ^3.1.0+3
  riverpod_analyzer_utils: ^0.1.2
```

---

## 2. Geliştirme Fazları ve Süreç Takvimi (Sprints)

Proje, toplam **18 haftada** tamamlanacak şekilde 6 ana faza bölünmüştür. Her faz sonunda stabil bir "Milestone Release" hedeflenir.

```
+----------------------------------------------------------------------------------+
| F1: Altyapı (H1-3) -> F2: Firestore (H4-7) -> F3: Auth (H8-10) -> F4: Storage (H11-13) |
|                                                                                  |
|                      F5: Scripting (H14-16) -> F6: Test & Release (H17-18)       |
+----------------------------------------------------------------------------------+
```

### FAZ 1: Altyapı, Yerel Depolama ve Workspace Yönetimi (Hafta 1 - 3)
- **Hafta 1**: Flutter projesinin kurulması, klasör yapısının oluşturulması, temel tasarım temalarının entegre edilmesi.
- **Hafta 2**: Isar yerel veritabanı şemalarının kurulması ve kod oluşturucunun çalıştırılması. `flutter_secure_storage` entegrasyonu.
- **Hafta 3**: Google OAuth 2.0, Service Account JSON yükleme ve Firebase CLI dosyalarını okuma servislerinin kodlanması. Workspace oluşturma, renklendirme ve "Salt Okunur" modlarının arayüze bağlanması.

### FAZ 2: Firestore Explorer ve Sorgu Motoru (Hafta 4 - 7)
- **Hafta 4**: Koleksiyon ve döküman gezinme ağacının (Sidebar & Navigation Tree) tasarımı ve kodlanması. gRPC canlı dinleme (realtime listening) servislerinin entegrasyonu.
- **Hafta 5**: Tablo Görünümü (Table View) ve Ağaç Görünümü (Tree View) geliştirmeleri. Satır içi (inline) düzenleme yeteneklerinin eklenmesi.
- **Hafta 6**: JSON Görünümü ve entegrasyonu. Gelişmiş Sorgu Oluşturucu (Query Builder) arayüzünün yapılması ve sorgu kaydetme (Saved Query) lojiğinin kurulması.
- **Hafta 7**: Geopoint harita entegrasyonu ve Firestore resim önizleme bileşenlerinin yazılması. Döküman kopyalama/taşıma ve güvenli yeniden adlandırma (Rename) işlemlerinin tamamlanması.

### FAZ 3: Firebase Authentication Yönetici Paneli (Hafta 8 - 10)
- **Hafta 8**: Auth kullanıcı listesi, e-posta/UID arama ve tarih sıralama arayüzünün kodlanması.
- **Hafta 9**: Kullanıcı detay paneli, şifre sıfırlama, e-posta değiştirme, hesap dondurma ve silme aksiyonlarının servis katmanıyla birleştirilmesi.
- **Hafta 10**: Custom Claims JSON editörünün yapılması. CSV/JSON formatında Auth kullanıcılarını içe ve dışa aktarma (Import/Export) süreçlerinin yazılması.

### FAZ 4: Firebase Storage Entegrasyonu ve Medya Oynatıcılar (Hafta 11 - 13)
- **Hafta 11**: Storage klasör gezgini, dosya listeleme, silme ve arama arayüzlerinin oluşturulması. Klasör simülasyon mekanizmasının (.placeholder) kodlanması.
- **Hafta 12**: Sürükle-bırak (Drag and Drop) desteği. İndirme ve Yükleme Kuyruğu (Queue Manager) tasarımı; eşzamanlı transfer kontrolörü, duraklat/devam et lojiğinin entegrasyonu.
- **Hafta 13**: Entegre video, ses ve resim oynatıcı panellerinin yazılması. Dosya metadata (MIME, Cache-Control, Custom) düzenleyicisinin tamamlanması.

### FAZ 5: JS Scripting Shell ve Veri Transfer Araçları (Hafta 14 - 16)
- **Hafta 14**: `flutter_js` sanal makinesinin entegrasyonu. Dart tarafındaki Firestore/Auth API servislerini JS VM'e bind eden köprünün (Bridge) yazılması.
- **Hafta 15**: Sözdizimi renklendirmeli kod editörünün ve canlı konsol çıktı terminalinin (Console Log Viewer) yapılması. Sık kullanılan şablonların eklenmesi.
- **Hafta 16**: CSV/JSON veri aktarım sihirbazının (Alan eşleme ve tip dönüştürme) yapılması. Projeler arası canlı döküman taşıma (Migration Wizard) aracının tamamlanması.

### FAZ 6: Test, Optimizasyon ve Platform Sürümleri (Hafta 17 - 18)
- **Hafta 17**: Tüm modüllerin entegrasyon testleri. Performans optimizasyonları (Büyük listelerde `ListView.builder` ve bellek yönetimi). Hata ayıklama ve loglama sisteminin kontrolü.
- **Hafta 18**: İşletim sistemlerine özel (macOS, Windows, Linux, iOS, Android) paketleme, kod imzalama (signing) ve mağaza süreçlerinin tamamlanması.

---

## 3. Test Stratejisi

Güvenli ve hatasız bir çalışma için 3 aşamalı test planı uygulanır:

1. **Birim Testleri (Unit Tests - Dart & Riverpod)**:
   - Tüm veri modellerinin (DTO) JSON serileştirme testleri.
   - Riverpod state notifier sınıflarının iş mantığı testleri (Mock veritabanı ve mock API servisleri kullanılarak).
   - Güvenli depolama (`flutter_secure_storage`) okuma-yazma testleri.
2. **Entegrasyon Testleri (Integration Tests - Firebase Emulator)**:
   - Yerel Firebase Emulator Suite'e bağlanarak Firestore döküman oluşturma/güncelleme/silme senaryoları.
   - JS Scripting VM'in emülatör üzerinde başarıyla veri okuyup yazabildiğinin doğrulanması.
   - Storage dosya yükleme ve indirme kuyruğunun veri bütünlüğü testleri.
3. **Kullanıcı Kabul Testleri (UAT)**:
   - Büyük veri setleri (100,000+ döküman) ile tablo görünümünün performans ve bellek (Memory Leak) testleri.
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
