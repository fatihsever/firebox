# Lisanslama, Ödeme ve Manuel Doğrulama Altyapısı - Firebox

Bu doküman, Firebox uygulamasının ticari dağıtımı için kurgulanan gelir modelini, masaüstü ve mobil platformlar arasındaki manuel lisans anahtarı doğrulamalı çapraz platform senkronizasyon mimarisini, Stripe/Paddle ve Apple/Google mobil ödeme sistemlerinin (Adapty) entegrasyonunu, basitleştirilmiş kurumsal toplu lisanslama altyapısını ve yerel güvenlik doğrulamalarını teknik detaylarıyla açıklamaktadır.

Firebox, **"Sıfır Sunucu Maliyeti / Maksimum Güvenlik"** prensibine sadık kalmak amacıyla bünyesinde herhangi bir kullanıcı kayıt, üye oturumu açma veya hesap yönetim arayüzü barındırmaz. Lisans doğrulama işlemleri tamamen manuel girilen kriptografik **Lisans Anahtarı** tabanlı çalışır. Lisans sunucusu ise tamamen arka planda (headless) sessizce çalışarak satın alma webhook'larını işler ve lisans anahtarlarını kullanıcılara e-posta ile ulaştırır.

---

## 1. Gelir Modeli ve Lisanslama Yapısı (Revenue Model)

Firebox, hem bireysel geliştiricileri hem de geniş yazılım ekiplerini (Enterprise) hedefleyen, esnek ödeme seçeneklerine sahip bir **SaaS (Software-as-a-Service)** modeliyle lisanslanır.

### 1.1. Fiyatlandırma Katmanları (Pricing Tiers)
Uygulama, yeni kullanıcıların tüm özellikleri değerlendirebilmesi için **7 Günlük Ücretsiz Deneme (Trial)** süresi sunar. Bu süre dolduktan sonra uygulamanın tüm profesyonel yeteneklerinden faydalanabilmek için **Pro** veya **Enterprise** katmanlarından birinin lisans anahtarıyla etkinleştirilmesi gerekir.

| Özellik | 7 Günlük Deneme (Trial) | Pro (Bireysel) | Enterprise (Kurumsal) |
| :--- | :--- | :--- | :--- |
| **Fiyat** | $0 (7 Günlük Deneme) | $9 / Aylık veya $89 / Yıllık | Kişi başı $15 / Aylık veya Özel Teklif |
| **Ömür Boyu Seçeneği** | Yok | $199 (Tek Seferlik Ödeme) | Yok |
| **Aktif Workspace Sınırı** | Sınırsız (Deneme süresince) | Sınırsız | Sınırsız |
| **Firestore Görünümü** | Tablo, Ağaç, JSON (Sınırsız) | Tablo, Ağaç, JSON (Sınırsız) | Tablo, Ağaç, JSON (Sınırsız) |
| **JS Scripting Shell** | Açık (Deneme süresince) | Açık | Açık |
| **DuckDB Yerel Analiz & SQL** | Açık (Sınırsız) | Açık (Sınırsız) | Açık (Sınırsız) |
| **Storage Transfer Kuyruğu** | Sınırsız / Sürükle-Bırak | Sınırsız / Sürükle-Bırak | Sınırsız / Sürükle-Bırak |
| **Çapraz Platform Kullanımı** | Evet (Yalnızca kurulan cihazda) | Evet (2 aktif cihaz: 1 Desktop, 1 Mobil) | Evet (Cihaz sınırı kurumsal belirlenir) |
| **Toplu Lisanslama** | Yok | Yok | Evet (Çoklu koltuk kapasiteli tek bir lisans anahtarı) |
| **Öncelikli Destek** | E-posta Desteği | E-posta Desteği | 7/24 Slack / E-posta SLA |

---

## 2. Çapraz Platform Lisans Senkronizasyon Mimarisi (Cross-Platform License Sync)

Kullanıcıların masaüstünden (Stripe/Paddle) satın aldıkları lisansları mobil uygulamada, mobilden (App Store/Google Play) satın aldıkları abonelikleri ise masaüstünde anında ve sorunsuz kullanabilmeleri için **merkezi bir Firebox Lisans Doğrulama Sunucusu (Firebox Licensing Server)** tasarlanmıştır. Bu sunucu kullanıcı dostu bir arayüze/panele sahip değildir; sadece arka planda API olarak çalışır ve başarılı ödemelerde lisans bilgilerini e-posta ile gönderir.

### 2.1. Lisans Senkronizasyon Akışı
İstemci (Client-only) local-first prensibiyle çalışırken, lisanslama için güvenli bir JWT (JSON Web Token) tabanlı bulut doğrulama katmanı entegre edilir. Bu sunucu, **Supabase Edge Functions** veya **Node.js/Firebase Cloud Functions** üzerinde hafif bir mikroservis olarak çalıştırılır.

```
+-----------------------------------------------------------------------------+
|                               FIREBOX CLIENT                                |
|                                                                             |
|  1. Kullanıcı Lisans Anahtarını Manuel Girer                                |
|  2. Aktivasyon Talebi gönderir (Key + Device ID) ----------\                |
|  5. İmzalı JWT alır (License, Expiry, Sign) <-----------------\               |
|  6. JWT'yi Secure Storage ve DuckDB'ye yazar                                |
|                                                                             |
+---------------------------------------------------------------\-------------+
                                                                \ (Secure HTTPS)
                                                                 v
+-----------------------------------------------------------------------------+
|                         FIREBOX LICENSING SERVER                            |
|                                                                             |
|  - Lisans veritabanını ve cihaz kayıtlarını yönetir (No-UI, headless).      |
|  - Satın alma webhook'larını işler (Stripe, Adapty, vb.)                    |
|  - Başarılı ödemelerde Lisans Anahtarını e-posta ile gönderir.              |
|  - 3. Güvenli token (JWT) imzalayıp istemciye döner.                        |
+-----------------------------------------------------------------------------+
       ^                                                               ^
       | (Webhooks)                                                    | (Webhooks)
+-------------------------+                                     +-------------+
|     STRIPE / PADDLE     |                                     |   ADAPTY    |
|  (Masaüstü Satın Alma)   |                                     | (Mobil IAP) |
+-------------------------+                                     +-------------+
```

### 2.2. Lisans Doğrulama Adımları (Step-by-Step Portability)

#### Senaryo A: Masaüstünden Satın Alma ve Mobilde Etkinleştirme
1. Kullanıcı web sitesi üzerinden Stripe veya Paddle entegrasyonuyla ödeme yapar.
2. Stripe, satın alım başarılı olduğunda **Firebox Licensing Server**'a bir webhook (`invoice.paid` veya `checkout.session.completed`) gönderir.
3. Sunucu otomatik olarak Stripe faturasındaki e-posta adresi için benzersiz bir kriptografik **Lisans Anahtarı** (`FBX-PRO-XXXX-XXXX-XXXX`) üretir ve bu anahtarı müşteriye e-posta ile gönderir (SendGrid, Mailgun vb. aracılığıyla).
4. Kullanıcı Firebox mobil uygulamasını açar, "Lisans Anahtarı Gir" ekranına bu anahtarı manuel yazar.
5. Mobil uygulama, Licensing Server'dan `/api/v1/license/activate` uç noktasını cihazın benzersiz ID'si ile asenkron olarak sorgular. Sunucu, SHA-256 ile imzalanmış, süresi ve yetki sınırları belirlenmiş bir JWT döndürür.
6. Mobil uygulama JWT'yi doğrular, Premium özellikleri açar ve JWT'yi cihazın yerel **Keystore/Keychain** katmanına ve DuckDB'ye kaydeder.

#### Senaryo B: Mobilden Satın Alma ve Masaüstünde Etkinleştirme
1. Kullanıcı iOS App Store veya Google Play Store üzerinden uygulama içi satın alma (IAP) ile abonelik başlatır.
2. Mobil uygulama, ödeme doğrulamasını **Adapty SDK**'sı aracılığıyla yürütür ve mobil premium özellikleri anında aktifleştirir.
3. Mobil uygulamadaki "Masaüstünde de Kullan" butonuna tıklayan kullanıcıya e-posta adresi sorulur.
4. Kullanıcı e-postasını girdiğinde, mobil uygulama Lisans Sunucusuna `/api/v1/license/generate-from-mobile` isteği gönderir. Bu istekte e-posta ve Adapty Kullanıcı Kimliği iletilir.
5. Lisanslama Sunucusu, Adapty API üzerinden bu mobil aboneliği doğrular, bir **Lisans Anahtarı** üretir ve kullanıcının girdiği e-postaya gönderir.
6. Kullanıcı bu anahtarı masaüstü uygulamasına manuel olarak girer ve masaüstündeki premium özellikleri de aktif eder.

---

## 3. Ödeme ve Entegrasyon Katmanları (Payment Gateways)

### 3.1. Masaüstü Entegrasyonu: Stripe & Paddle
Masaüstü uygulamasında (Mac, Windows, Linux) yerel olarak kredi kartı almak veya harici bir ödeme sayfasına yönlendirmek için **Stripe / Paddle** yapısı kurulur.

- **Stripe Checkout (Yönlendirmeli Ödeme)**: Kullanıcı lisans satın almak istediğinde, uygulama `url_launcher` kullanarak Stripe Checkout sayfasına yönlendirir. Stripe sayfasında ödeme tamamlandıktan sonra kullanıcı deep link (`firebox://`) ile uygulamaya geri döndürülür.
- **Webhook Entegrasyonu**: Stripe'tan gelen ödeme tamamlandı webhook'u sonrasında arka planda çalışan sunucumuz lisans anahtarını oluşturup müşteriye e-posta ile otomatik ulaştırır.

### 3.2. Mobil Entegrasyon: Adapty (App Store & Google Play)
Apple App Store ve Google Play Store'un karmaşık abonelik süreçlerini, fatura iadelerini, deneme sürelerini ve çapraz platform doğrulamalarını tek bir merkezden yönetmek için **Adapty (`adapty_flutter`)** paketi kullanılır.

- **Adapty Entegrasyon Akışı**:
  - Mobil uygulama içinden Adapty paywall'u tetiklenir ve yerel ödeme ekranı açılır.
  - Satın alım sonrasında Adapty Webhook'u Lisanslama Sunucumuzu tetikleyerek mobil aboneliği kaydeder.
  - Mobil uygulamada kullanıcı e-posta adresini girerek "Masaüstü Lisans Anahtarını Gönder" talebi oluşturabilir, böylece sunucumuz mobil aboneliğe bağlı lisans anahtarını üretip e-posta ile ulaştırır.

---

## 4. Kurumsal ve Toplu Lisanslama Altyapısı (Enterprise & Bulk Licensing)

Büyük yazılım ekipleri, geliştirme ajansları ve kurumsal şirketler için uygulama içinde karmaşık dairesel döküm grafikleri, davet formları, koltuk iptal etme panelleri gibi şişkinlik (bloat) yaratacak arayüzlere yer verilmez. Süreç son derece basit ve sunucu tabanlı olarak yürütülür.

### 4.1. Çoklu Koltuklu Tek Lisans Anahtarı (Seats Management)
1. **Manuel Toplu Satın Alım**: Şirketler toplu satın alımı (Örn: 50 Koltuk) satış ekibimiz veya Stripe üzerinden manuel olarak gerçekleştirir.
2. **Kapasite Ataması**: Lisans sunucumuz, bu şirkete özel tek bir Lisans Anahtarı (`FBX-ENT-XXXX-XXXX-XXXX`) oluşturur ve bu anahtarın cihaz aktivasyon limitini (Seats) 50 olarak ayarlar.
3. **Manuel Dağıtım**: Şirket yöneticisi bu tek anahtarı kendi çalışanlarına dağıtır. Her çalışan uygulamayı kurup aynı lisans anahtarını kendi cihazına manuel girer.
4. **Sunucu Tabanlı Limit Takibi**: Lisans sunucusu, bu anahtarla aktifleştirilen aktif cihaz sayısını sayar. Eğer 50 limitine ulaşılmışsa, yeni bir çalışanın aktivasyon isteğini reddeder. Ekipten ayrılan personelin koltuğu, arka planda sunucu üzerinden veya destek birimimiz aracılığıyla pasifleştirilir.

---

## 5. Yerel Veritabanı ve Güvenlik Tasarımı (Licensing DB & Security)

Firebox, lisans doğrulamasını güvenli kılmak ve korsan yazılım kullanımını engellemek amacıyla cihaz üzerinde çok katmanlı bir kontrol mekanizması işletir.

### 5.1. DuckDB SQL Lisans Tablosu (Local Cache)
Uygulamanın lisans durumunu yerel olarak hızlıca okuyabilmesi ve internet olmadığında da çalışabilmesi (Offline Mode) için DuckDB'de lisans tablosu kurgulanmıştır.

```sql
-- Lisans Bilgileri Tablosu
CREATE TABLE IF NOT EXISTS licensing_info (
    id INTEGER PRIMARY KEY DEFAULT 1,            -- Tek bir lisans olacağı için sabit id (1) verilir
    license_key VARCHAR NOT NULL,                -- Kullanıcının manuel girdiği lisans anahtarı
    registered_email VARCHAR,                    -- Lisansın bağlı olduğu e-posta adresi
    license_type VARCHAR NOT NULL,               -- 'trial', 'pro_monthly', 'pro_yearly', 'pro_lifetime', 'enterprise'
    license_status VARCHAR NOT NULL,             -- 'active', 'expired', 'canceled', 'revoked'
    activated_at TIMESTAMP NOT NULL,             -- Cihazda etkinleştirilme tarihi
    expires_at TIMESTAMP,                        -- Lisans bitiş tarihi (Lifetime için null)
    last_verified_at TIMESTAMP NOT NULL,         -- Sunucuyla yapılan son başarılı doğrulama tarihi
    signed_jwt VARCHAR NOT NULL                  -- Sunucudan gelen, içeriği değiştirilemez imzalı lisans token'ı
);
```

### 5.2. Hassas Bilgilerin Secure Storage'da Saklanması
DuckDB'deki `signed_jwt` ve `license_key` manipülasyonu önlemek amacıyla **`flutter_secure_storage`** içerisinde işletim sisteminin kriptolu kasasında (macOS Keychain, Windows Credential Manager, Android Keystore, iOS Keychain) saklanır. DuckDB'deki tablo sadece hızlı UI okumaları ve istatistikler için önbellek görevi görür. Uygulama açılışında her zaman Secure Storage'daki JWT çözülerek imzası kontrol edilir.

### 5.3. Çevrimdışı Esneklik Süresi (Offline Grace Period)
Geliştiricilerin internet bağlantısı olmayan ortamlarda (Örn: Uçakta, tünelde veya saha çalışmalarında) Firebox'ı kullanmaya devam edebilmeleri için **7 günlük Çevrimdışı Esneklik Süresi** tanınır.

- **Doğrulama Algoritması**:
  - Uygulama açıldığında, DuckDB ve Secure Storage'dan `last_verified_at` tarihini ve JWT'yi okur.
  - Eğer `Şimdiki Zaman - last_verified_at < 7 Gün` ise, internet olmasa dahi JWT içerisindeki imza yerel olarak doğrulanarak kullanıcının premium özellikleri kullanmasına izin verilir.
  - Eğer süre 7 günü aşmışsa ve internet bağlantısı yoksa, uygulama kullanıcıdan internete bağlanmasını ve lisansını doğrulatmasını ister. Lisans doğrulanamazsa, uygulama tüm özellikleri kilitler ve lisans anahtarı giriş ekranını gösterir.

---

## 6. Kod Tasarımı ve Riverpod State Yönetimi

### 6.1. Dart Lisans Servisi Sınıfı (`LicensingService`)

Geliştiricilerin hem mobil (Adapty) hem de masaüstü (Stripe/Custom API) süreçlerini tek bir çatı altından manuel lisans anahtarıyla yönetmesini sağlayan servis tasarımı aşağıda sunulmuştur:

```dart
// lib/core/licensing/licensing_service.dart
import 'dart:convert';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:dart_duckdb/dart_duckdb.dart';
import 'package:http/http.dart' as http;

enum LicenseType { trial, proMonthly, proYearly, proLifetime, enterprise }
enum LicenseStatus { active, expired, canceled, revoked }

class LicensingService {
  final FlutterSecureStorage _secureStorage;
  final Connection _dbConn;
  final String _licensingApiUrl = "https://api.firebox.to/v1/licensing";

  LicensingService({
    required FlutterSecureStorage secureStorage,
    required Connection dbConn,
  }) : _secureStorage = secureStorage, _dbConn = dbConn;

  /// Kullanıcının mevcut lisans durumunu önce yerelden yükler, internet varsa sunucuyla senkronize eder.
  Future<Map<String, dynamic>> checkLicenseStatus() async {
    // 1. Secure Storage'dan JWT ve Lisans Anahtarını oku
    final jwt = await _secureStorage.read(key: "firebox_license_jwt");
    final licenseKey = await _secureStorage.read(key: "firebox_license_key");
    final trialStartDateStr = await _secureStorage.read(key: "firebox_trial_start_date");

    // Eğer lisans girilmemişse, 7 günlük deneme süresini kontrol et
    if (jwt == null || licenseKey == null) {
      if (trialStartDateStr == null) {
        final nowStr = DateTime.now().toIso8601String();
        await _secureStorage.write(key: "firebox_trial_start_date", value: nowStr);
        return {
          "type": LicenseType.trial,
          "status": LicenseStatus.active,
          "trialDaysLeft": 7
        };
      } else {
        final trialStart = DateTime.parse(trialStartDateStr);
        final diffDays = DateTime.now().difference(trialStart).inDays;
        final daysLeft = 7 - diffDays;
        if (daysLeft > 0) {
          return {
            "type": LicenseType.trial,
            "status": LicenseStatus.active,
            "trialDaysLeft": daysLeft
          };
        } else {
          return {
            "type": LicenseType.trial,
            "status": LicenseStatus.expired,
            "trialDaysLeft": 0
          };
        }
      }
    }

    // 2. JWT imzasını ve yerel süre doğrulamalarını yap
    final payload = _decodeJwtPayload(jwt);
    final expiresAt = payload["exp"] != null ? DateTime.parse(payload["exp"]) : null;
    final lastVerified = DateTime.parse(payload["last_verified"]);

    // Çevrimdışı kontrolü (Grace Period - 7 Gün)
    if (DateTime.now().difference(lastVerified).inDays < 7) {
      if (expiresAt == null || expiresAt.isAfter(DateTime.now())) {
        return {
          "type": _parseLicenseType(payload["license_type"]),
          "status": LicenseStatus.active,
          "licenseKey": licenseKey,
          "registeredEmail": payload["email"],
          "jwt": jwt
        };
      }
    }

    // 3. Süre aşılmışsa veya online doğrulama gerekiyorsa sunucuya sor (İnternet varsa)
    try {
      final response = await http.post(
        Uri.parse("$_licensingApiUrl/verify"),
        headers: {"Content-Type": "application/json"},
        body: jsonEncode({
          "license_key": licenseKey,
          "device_id": await _getDeviceUniqueId(),
          "jwt": jwt,
        }),
      ).timeout(const Duration(seconds: 5));

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        final newJwt = data["jwt"];

        // Yeni JWT'yi yerel sistemlere kaydet
        await _saveLicenseToLocal(licenseKey, newJwt, data);

        return {
          "type": _parseLicenseType(data["license_type"]),
          "status": _parseLicenseStatus(data["status"]),
          "licenseKey": licenseKey,
          "registeredEmail": data["email"],
          "jwt": newJwt
        };
      }
    } catch (_) {
      // İnternet hatası durumunda, eğer yerel expire süresi dolmadıysa eski durumu koru
      if (expiresAt == null || expiresAt.isAfter(DateTime.now())) {
        return {
          "type": _parseLicenseType(payload["license_type"]),
          "status": LicenseStatus.active,
          "licenseKey": licenseKey,
          "registeredEmail": payload["email"],
          "jwt": jwt
        };
      }
    }

    return {"type": LicenseType.trial, "status": LicenseStatus.expired, "trialDaysLeft": 0};
  }

  /// Manuel Lisans Anahtarı ile Aktivasyon Yapma
  Future<bool> activateWithLicenseKey(String licenseKey) async {
    try {
      final response = await http.post(
        Uri.parse("$_licensingApiUrl/activate"),
        headers: {"Content-Type": "application/json"},
        body: jsonEncode({
          "license_key": licenseKey,
          "device_id": await _getDeviceUniqueId(),
          "platform": _getPlatformName(),
        }),
      ).timeout(const Duration(seconds: 10));

      if (response.statusCode == 200) {
        final data = jsonDecode(response.body);
        await _saveLicenseToLocal(licenseKey, data["jwt"], data);
        return true;
      }
    } catch (_) {
      return false;
    }
    return false;
  }

  /// Mobil satın alım sonrasında e-posta ile lisans anahtarı talep etme/köprü oluşturma
  Future<bool> requestLicenseKeyForMobilePurchase(String email, String adaptyUserId) async {
    try {
      final response = await http.post(
        Uri.parse("$_licensingApiUrl/generate-from-mobile"),
        headers: {"Content-Type": "application/json"},
        body: jsonEncode({
          "email": email,
          "adapty_user_id": adaptyUserId,
        }),
      ).timeout(const Duration(seconds: 10));

      return response.statusCode == 200;
    } catch (_) {
      return false;
    }
  }

  // --- YARDIMCI METOTLAR ---

  Future<void> _saveLicenseToLocal(String licenseKey, String jwt, Map<String, dynamic> data) async {
    // 1. Secure Storage'a kaydet
    await _secureStorage.write(key: "firebox_license_key", value: licenseKey);
    await _secureStorage.write(key: "firebox_license_jwt", value: jwt);

    // 2. DuckDB SQL Cache güncellemesi
    final stmt = _dbConn.prepare('''
      INSERT OR REPLACE INTO licensing_info (
        id, license_key, registered_email, license_type, license_status,
        activated_at, expires_at, last_verified_at, signed_jwt
      ) VALUES (1, ?, ?, ?, ?, ?, ?, ?, ?);
    ''');

    stmt.execute([
      licenseKey,
      data["email"],
      data["license_type"],
      data["status"],
      DateTime.now().toIso8601String(),
      data["expires_at"], // Null olabilir (Lifetime)
      DateTime.now().toIso8601String(),
      jwt
    ]);
    stmt.dispose();
  }

  Map<String, dynamic> _decodeJwtPayload(String jwt) {
    final parts = jwt.split('.');
    if (parts.length != 3) {
      throw Exception("Geçersiz JWT formatı");
    }
    final payloadPart = parts[1];
    final normalized = base64Url.normalize(payloadPart);
    final String resp = utf8.decode(base64Url.decode(normalized));
    return jsonDecode(resp);
  }

  LicenseType _parseLicenseType(String type) {
    switch (type) {
      case 'pro_monthly': return LicenseType.proMonthly;
      case 'pro_yearly': return LicenseType.proYearly;
      case 'pro_lifetime': return LicenseType.proLifetime;
      case 'enterprise': return LicenseType.enterprise;
      default: return LicenseType.trial;
    }
  }

  LicenseStatus _parseLicenseStatus(String status) {
    switch (status) {
      case 'active': return LicenseStatus.active;
      case 'canceled': return LicenseStatus.canceled;
      case 'revoked': return LicenseStatus.revoked;
      default: return LicenseStatus.expired;
    }
  }

  Future<String> _getDeviceUniqueId() async {
    return "DEVICE_HARDWARE_ID_12345"; // Platforma özel UUID okuma lojiği
  }

  String _getPlatformName() {
    return "desktop";
  }
}
```

### 6.2. Riverpod Lisans Durum Yönetimi (`licensingStateProvider`)

Uygulama arayüzünün premium yetenekleri açıp kapatırken dinleyeceği asenkron Riverpod sağlayıcısı:

```dart
// lib/core/licensing/licensing_provider.dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'licensing_service.dart';

class LicensingState {
  final LicenseType type;
  final LicenseStatus status;
  final String? licenseKey;
  final String? registeredEmail;
  final bool isLoading;
  final String? errorMessage;

  bool get isPremium =>
      status == LicenseStatus.active;

  bool get isEnterprise =>
      status == LicenseStatus.active && type == LicenseType.enterprise;

  LicensingState({
    required this.type,
    required this.status,
    this.licenseKey,
    this.registeredEmail,
    this.isLoading = false,
    this.errorMessage,
  });

  LicensingState copyWith({
    LicenseType? type,
    LicenseStatus? status,
    String? licenseKey,
    String? registeredEmail,
    bool? isLoading,
    String? errorMessage,
  }) {
    return LicensingState(
      type: type ?? this.type,
      status: status ?? this.status,
      licenseKey: licenseKey ?? this.licenseKey,
      registeredEmail: registeredEmail ?? this.registeredEmail,
      isLoading: isLoading ?? this.isLoading,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }
}

class LicensingNotifier extends StateNotifier<LicensingState> {
  final LicensingService _service;

  LicensingNotifier(this._service)
      : super(LicensingState(type: LicenseType.trial, status: LicenseStatus.expired)) {
    checkLicense();
  }

  /// Uygulama açılışında lisans durumunu doğrular
  Future<void> checkLicense() async {
    state = state.copyWith(isLoading: true);
    try {
      final result = await _service.checkLicenseStatus();
      state = LicensingState(
        type: result["type"] as LicenseType,
        status: result["status"] as LicenseStatus,
        licenseKey: result["licenseKey"] as String?,
        registeredEmail: result["registeredEmail"] as String?,
        isLoading: false,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        errorMessage: "Lisans doğrulaması başarısız: ${e.toString()}",
      );
    }
  }

  /// Lisans Anahtarı ile Etkinleştirme
  Future<bool> activateKey(String licenseKey) async {
    state = state.copyWith(isLoading: true);
    final success = await _service.activateWithLicenseKey(licenseKey);
    if (success) {
      await checkLicense();
      return true;
    } else {
      state = state.copyWith(
        isLoading: false,
        errorMessage: "Geçersiz veya limiti aşılmış lisans anahtarı.",
      );
      return false;
    }
  }
}
```

---

## 7. UI/UX Ödeme ve Paywall Tasarım İlkeleri

Kullanıcıların satın alma deneyimini maksimize etmek ve arayüzde premium hissi uyandırmak amacıyla şu kurallar uygulanır:

1. **Doğal Limit Uyarıları (Soft Paywalls)**:
   - Kullanıcı deneme süresi bittiğinde veya lisanssız kullanımda kısıtlı bir işlem yapmaya çalıştığında (Örn: Bir Proje/Workspace eklemeye çalışırken), sert bir hata ekranı yerine, şık bir modal panel ile Pro özelliklerin avantajları anlatılarak doğrudan satın alma linklerine yönlendirilir.
2. **Premium Göstergeleri**:
   - Lisans aktif olduğunda, kullanıcı profilinin yanında şık, altın/mor renkli bir **\"PRO\"** veya **\"ENTERPRISE\"** rozeti (badge) gösterilir.
   - Pro özelliklerin yanına (Örn: Scripting Shell, DuckDB Analytics) hafif ve şık kilit simgeleri yerleştirilir. Lisanssız veya deneme süresi dolmuş kullanıcılarda bu alanlar pasif (disabled) durur ve üzerine tıklandığında manuel lisans giriş penceresi açılır.
3. **Manuel Lisans Giriş Ekranı (Manual Key Entry View)**:
   - Kullanıcıların lisans anahtarlarını girebilecekleri ekran son derece sade tasarlanır.
   - Ekran üzerinde büyük, monospaced fontla yazılmış bir metin kutusu (`FBX-PRO-XXXX-XXXX-XXXX`), bir "Yapıştır" butonu, bir "Etkinleştir" butonu yer alır.
   - Kutunun hemen altında yer alan açıklama: *"Satın aldığınız lisans anahtarı faturanıza bağlı e-posta adresinize gönderilmiştir. Lisansınızı aynı anda en fazla 2 aktif cihazda (Maks. 1 Masaüstü ve Maks. 1 Mobil) manuel etkinleştirebilirsiniz. Aynı lisans aynı anda 2 masaüstü veya 2 mobil cihazda çalışamaz."*
4. **Mobil "Masaüstünde de Kullan" Köprüsü**:
   - Mobil uygulamadan Adapty aracılığıyla abone olan kullanıcılara, ayarlarda "Masaüstünde de Kullan" seçeneği gösterilir.
   - Kullanıcı buraya e-postasını yazdığında sunucuya giden istek doğrulanır ve kullanıcının e-postasına taşınabilir Lisans Anahtarı gönderilir. Böylece kullanıcı masaüstü uygulamasına bu anahtarı yazarak lisansını iki platformda da birleştirmiş olur.
