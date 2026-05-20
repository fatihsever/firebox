# Veri Modelleri ve Yerel Depolama - Firebox

Bu doküman, Firebox uygulamasının yerel veritabanında (**DuckDB**) tutacağı verilerin ilişkisel şemalarını, uygulama içi kullanılan veri modellerini, Firebase API'leri ile haberleşirken kullanılan DTO (Data Transfer Object) yapılarını ve Dart dilinde DuckDB entegrasyon kodlarını tanımlamaktadır.

---

## 1. Yerel Veritabanı Şemaları (DuckDB SQL Schemas)

Yerel veritabanı, kullanıcının uygulama ayarlarını, bağlantı profillerini, geçmiş işlemlerini, **Firestore yerel yedeklerini (local-first cache/backup)** ve **Lisans/Abonelik durumlarını** saklamak amacıyla yüksek performanslı ilişkisel analitik veritabanı olan **DuckDB** kullanılarak kurgulanmıştır.

Aşağıda, uygulamanın ilk açılışında çalıştırılacak SQL Tablo Tanımları (DDL) ve ilgili açıklamaları yer almaktadır.

### 1.1. `workspaces` (Çalışma Alanı / Proje Bağlantısı)
Kullanıcının kaydettiği her bir Firebase projesini temsil eder. Hassas kimlik doğrulama verileri (Service Account özel anahtarları veya Google OAuth Refresh Token'lar) doğrudan bu tabloda değil, `flutter_secure_storage` içerisinde şifreli olarak saklanır. Bu tabloda sadece konfigürasyon ve referans bilgileri tutulur.

```sql
CREATE TABLE IF NOT EXISTS workspaces (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    project_id VARCHAR UNIQUE NOT NULL,      -- Firebase Proje Kimliği (Örn: firebox-prod-123)
    name VARCHAR NOT NULL,                    -- Kullanıcının verdiği ad (Örn: Canlı Sunucu)
    color_hex VARCHAR NOT NULL DEFAULT '#6366F1', -- UI'da kullanılacak tema rengi (Örn: #EF4444)
    is_read_only BOOLEAN NOT NULL DEFAULT FALSE,  -- Yanlışlıkla yazmaları önlemek için salt okunur modu
    auth_method VARCHAR NOT NULL,             -- 'googleSignIn', 'serviceAccount', 'emulator', 'cli'
    service_account_file_name VARCHAR,        -- Servis hesabı dosya adı (Görsel bilgi için)
    firestore_emulator_port INTEGER,          -- Emülatör portları (Eğer auth_method = 'emulator' ise)
    auth_emulator_port INTEGER,
    storage_emulator_port INTEGER,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_accessed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 1.2. `saved_queries` (Kaydedilen Sorgular)
Firestore Explorer üzerinde kullanıcının tekrar tekrar kullanmak için kaydettiği gelişmiş GUI sorgularıdır.

```sql
CREATE TABLE IF NOT EXISTS saved_queries (\n    id INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id INTEGER NOT NULL,            -- Hangi projeye ait olduğu (FOREIGN KEY)
    name VARCHAR NOT NULL,                    -- Sorguya verilen ad (Örn: Aktif Premium Üyeler)
    collection_path VARCHAR NOT NULL,         -- Koleksiyon yolu (Örn: users veya orders/123/items)
    filters_json VARCHAR NOT NULL,            -- Filtre dizisinin JSON hali (Örn: [{"field":"age","op":">=","value":18}])
    order_by_field VARCHAR,                   -- Sıralama yapılacak alan adı
    order_by_descending BOOLEAN NOT NULL DEFAULT FALSE, -- Azalan sıralama mı?
    limit_val INTEGER,                        -- Çekilecek maksimum döküman sayısı
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (workspace_id) REFERENCES workspaces(id) ON DELETE CASCADE
);
```

### 1.3. `transfer_history` (Veri Taşıma Geçmişi)
Kullanıcının projeler arası yaptığı veri transferlerinin veya içe/dışa aktarım (Import/Export) işlemlerinin istatistiksel kaydı.

```sql
CREATE TABLE IF NOT EXISTS transfer_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    source_workspace_id INTEGER NOT NULL,
    target_workspace_id INTEGER,              -- Dosya dışa aktarımında boş olabilir
    transfer_type VARCHAR NOT NULL,           -- 'firestore', 'auth', 'storage'
    status VARCHAR NOT NULL,                  -- 'running', 'completed', 'failed', 'paused'
    processed_count INTEGER NOT NULL DEFAULT 0, -- Taşınan döküman/dosya sayısı
    total_count INTEGER NOT NULL DEFAULT 0,     -- Toplam döküman/dosya sayısı
    error_message VARCHAR,                    -- Varsa hata detayları
    started_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP,
    FOREIGN KEY (source_workspace_id) REFERENCES workspaces(id) ON DELETE CASCADE
);
```

### 1.4. `app_settings` (Uygulama Genel Ayarları)
Uygulamanın genel davranışını ve görünümünü belirleyen tek satırlık ayarlardır.

```sql
CREATE TABLE IF NOT EXISTS app_settings (
    id INTEGER PRIMARY KEY,                   -- Tek ayar satırı olacağı için sabit id (1) verilir
    theme_mode VARCHAR NOT NULL DEFAULT 'dark',  -- 'light', 'dark', 'system'
    language_code VARCHAR NOT NULL DEFAULT 'tr', -- 'tr', 'en'
    timestamp_format VARCHAR NOT NULL DEFAULT 'yyyy-MM-dd HH:mm:ss',
    default_map_provider VARCHAR NOT NULL DEFAULT 'openstreetmap',
    enable_autocomplete BOOLEAN NOT NULL DEFAULT TRUE
);
```

### 1.5. `firestore_local_cache` (Gelişmiş Analitik Raporlama ve Caching Tablosu)
Bu tablo, DuckDB'nin sunduğu yüksek performanslı analitik güçten yararlanmak için tasarlanmıştır. Kullanıcılar Firestore koleksiyonlarının anlık yedeklerini (Backups) yerel DuckDB veritabanına indirebilir ve bu veri üzerinde doğrudan SQL sorguları koşturabilir.

```sql
CREATE TABLE IF NOT EXISTS firestore_local_cache (
    id VARCHAR NOT NULL,                     -- Firestore döküman ID'si (UUID veya Özel ID)
    workspace_id INTEGER NOT NULL,            -- Hangi projeye ait olduğu
    collection_path VARCHAR NOT NULL,         -- Koleksiyon yolu (Örn: 'users', 'transactions')
    document_id VARCHAR NOT NULL,             -- Belge ID'si (Örn: 'user_99')
    fields_json JSON NOT NULL,                -- DuckDB yerel JSON veri tipi (Dökümanın tüm alanları)
    synced_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (workspace_id, collection_path, document_id),
    FOREIGN KEY (workspace_id) REFERENCES workspaces(id) ON DELETE CASCADE
);
```

### 1.6. `licensing_info` (Lisans ve Abonelik Bilgisi - YENİ)
Uygulamanın lisans durumunu cihaz üzerinde önbelleğe alarak hızlı UI kontrolleri ve 7 günlük çevrimdışı kullanım (Offline Grace Period) esnekliği sağlamak amacıyla tasarlanmıştır. Firebox'ta kullanıcı girişi bulunmadığından, lisans tamamen manuel girilen Lisans Anahtarı tabanlı olarak kontrol edilir.

```sql
CREATE TABLE IF NOT EXISTS licensing_info (
    id INTEGER PRIMARY KEY DEFAULT 1,         -- Tek bir lisans olacağı için her zaman 1
    license_key VARCHAR NOT NULL,             -- Kullanıcının manuel girdiği lisans anahtarı
    registered_email VARCHAR,                 -- Lisansın bağlı olduğu e-posta adresi (Örn: stripe faturası e-postası)
    license_type VARCHAR NOT NULL,            -- 'free', 'pro_monthly', 'pro_yearly', 'pro_lifetime', 'enterprise'
    license_status VARCHAR NOT NULL,          -- 'active', 'expired', 'canceled', 'revoked'
    activated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,                     -- Abonelik bitiş tarihi (Lifetime lisanslarda NULL'dur)
    last_verified_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, -- Sunucu ile yapılan son başarılı doğrulama
    signed_jwt VARCHAR NOT NULL               -- Sunucu tarafından SHA256 ile imzalanmış değiştirilemez token
);
```



---

## 2. Dart DuckDB Servis Katmanı Tasarımı

Firebox uygulamasında DuckDB veritabanı ile etkileşime girmek ve SQL sorguları çalıştırmak için kullanılacak olan Dart servis kod örneği aşağıda sunulmuştur. Bu servis `dart_duckdb` paketi bağımlılığı üzerine kurulmuştur ve lisanslama tablolarını da içerecek şekilde güncellenmiştir.

```dart
// lib/core/database/duckdb_service.dart
import 'dart:io';
import 'package:dart_duckdb/dart_duckdb.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';

class DuckDbService {
  late final Database _db;
  late final Connection _conn;
  bool _isInitialized = false;

  Connection get connection => _conn;

  /// Veritabanını başlatır, dosyayı açar ve tabloları oluşturur.
  Future<void> init() async {
    if (_isInitialized) return;

    // Uygulama destek dizinini al
    final appDir = await getApplicationSupportDirectory();
    final dbPath = p.join(appDir.path, 'firebox_local.db');

    // DuckDB Veritabanını aç / oluştur
    _db = Database(dbPath);
    _conn = Connection(_db);

    _isInitialized = true;

    // Tabloları oluştur (DDL)
    await _createTables();
    await _initSettings();
  }

  Future<void> _createTables() async {
    // 1. Workspaces Tablosu
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS workspaces (
        id INTEGER PRIMARY KEY,
        project_id VARCHAR UNIQUE NOT NULL,
        name VARCHAR NOT NULL,
        color_hex VARCHAR NOT NULL,
        is_read_only BOOLEAN NOT NULL,
        auth_method VARCHAR NOT NULL,
        service_account_file_name VARCHAR,
        firestore_emulator_port INTEGER,
        auth_emulator_port INTEGER,
        storage_emulator_port INTEGER,
        created_at TIMESTAMP NOT NULL,
        last_accessed_at TIMESTAMP NOT NULL
      );
    ''');

    // 2. Saved Queries Tablosu
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS saved_queries (
        id INTEGER PRIMARY KEY,
        workspace_id INTEGER NOT NULL,
        name VARCHAR NOT NULL,
        collection_path VARCHAR NOT NULL,
        filters_json VARCHAR NOT NULL,
        order_by_field VARCHAR,
        order_by_descending BOOLEAN NOT NULL,
        limit_val INTEGER,
        created_at TIMESTAMP NOT NULL
      );
    ''');

    // 3. Transfer History Tablosu
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS transfer_history (
        id INTEGER PRIMARY KEY,
        source_workspace_id INTEGER NOT NULL,
        target_workspace_id INTEGER,
        transfer_type VARCHAR NOT NULL,
        status VARCHAR NOT NULL,
        processed_count INTEGER NOT NULL,
        total_count INTEGER NOT NULL,
        error_message VARCHAR,
        started_at TIMESTAMP NOT NULL,
        ended_at TIMESTAMP
      );
    ''');

    // 4. App Settings Tablosu
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS app_settings (
        id INTEGER PRIMARY KEY,
        theme_mode VARCHAR NOT NULL,
        language_code VARCHAR NOT NULL,
        timestamp_format VARCHAR NOT NULL,
        default_map_provider VARCHAR NOT NULL,
        enable_autocomplete BOOLEAN NOT NULL
      );
    ''');

    // 5. Firestore Local Cache (Analizler için)
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS firestore_local_cache (
        workspace_id INTEGER NOT NULL,
        collection_path VARCHAR NOT NULL,
        document_id VARCHAR NOT NULL,
        fields_json VARCHAR NOT NULL,
        synced_at TIMESTAMP NOT NULL,
        PRIMARY KEY (workspace_id, collection_path, document_id)
      );
    ''');

    // 6. Licensing Info Tablosu (Yeni)
    _conn.execute('''
      CREATE TABLE IF NOT EXISTS licensing_info (
        id INTEGER PRIMARY KEY,
        license_key VARCHAR NOT NULL,
        registered_email VARCHAR,
        license_type VARCHAR NOT NULL,
        license_status VARCHAR NOT NULL,
        activated_at TIMESTAMP NOT NULL,
        expires_at TIMESTAMP,
        last_verified_at TIMESTAMP NOT NULL,
        signed_jwt VARCHAR NOT NULL
      );
    ''');
  }

  Future<void> _initSettings() async {
    final stmt = _conn.prepare('SELECT COUNT(*) FROM app_settings');
    final res = stmt.execute();
    final count = res.first.first as int;
    stmt.dispose();

    if (count == 0) {
      _conn.execute('''
        INSERT INTO app_settings (id, theme_mode, language_code, timestamp_format, default_map_provider, enable_autocomplete)
        VALUES (1, 'dark', 'tr', 'yyyy-MM-dd HH:mm:ss', 'openstreetmap', true);
      ''');
    }
  }

  // --- LİSANS CRUD OPERASYONLARI (YENİ) ---

  Future<void> updateLicensingLocalCache(Map<String, dynamic> license) async {
    final stmt = _conn.prepare('''
      INSERT OR REPLACE INTO licensing_info (
        id, license_key, registered_email, license_type, license_status, 
        activated_at, expires_at, last_verified_at, signed_jwt
      ) VALUES (1, ?, ?, ?, ?, ?, ?, ?, ?);
    ''');

    stmt.execute([
      license['licenseKey'],
      license['registeredEmail'],
      license['licenseType'],
      license['licenseStatus'],
      license['activatedAt'] ?? DateTime.now().toIso8601String(),
      license['expiresAt'], // Null olabilir (Lifetime)
      DateTime.now().toIso8601String(),
      license['signedJwt'],
    ]);
    stmt.dispose();
  }

  Future<Map<String, dynamic>?> getCachedLicensingInfo() async {
    final stmt = _conn.prepare('SELECT * FROM licensing_info WHERE id = 1');
    final res = stmt.execute();
    
    if (res.isEmpty) {
      stmt.dispose();
      return null;
    }

    final row = res.first;
    final Map<String, dynamic> license = {
      'licenseKey': row[1],
      'registeredEmail': row[2],
      'licenseType': row[3],
      'licenseStatus': row[4],
      'activatedAt': DateTime.parse(row[5] as String),
      'expiresAt': row[6] != null ? DateTime.parse(row[6] as String) : null,
      'lastVerifiedAt': DateTime.parse(row[7] as String),
      'signedJwt': row[8],
    };
    stmt.dispose();
    return license;
  }

  // --- WORKSPACE CRUD OPERASYONLARI ---

  Future<void> insertWorkspace(Map<String, dynamic> ws) async {
    final stmt = _conn.prepare('''
      INSERT INTO workspaces (
        project_id, name, color_hex, is_read_only, auth_method, 
        service_account_file_name, firestore_emulator_port, 
        auth_emulator_port, storage_emulator_port, created_at, last_accessed_at
      ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?);
    ''');

    stmt.execute([
      ws['projectId'],
      ws['name'],
      ws['colorHex'],
      ws['isReadOnly'] ? 1 : 0,
      ws['authMethod'],
      ws['serviceAccountFileName'],
      ws['firestoreEmulatorPort'],
      ws['authEmulatorPort'],
      ws['storageEmulatorPort'],
      DateTime.now().toIso8601String(),
      DateTime.now().toIso8601String(),
    ]);
    stmt.dispose();
  }

  Future<List<Map<String, dynamic>>> getAllWorkspaces() async {
    final stmt = _conn.prepare('SELECT * FROM workspaces ORDER BY last_accessed_at DESC');
    final res = stmt.execute();
    
    final List<Map<String, dynamic>> workspaces = [];
    for (final row in res) {
      workspaces.add({
        'id': row[0],
        'projectId': row[1],
        'name': row[2],
        'colorHex': row[3],
        'isReadOnly': row[4] == 1,
        'authMethod': row[5],
        'serviceAccountFileName': row[6],
        'firestoreEmulatorPort': row[7],
        'authEmulatorPort': row[8],
        'storageEmulatorPort': row[9],
        'createdAt': DateTime.parse(row[10] as String),
        'lastAccessedAt': DateTime.parse(row[11] as String),
      });
    }
    stmt.dispose();
    return workspaces;
  }

  // --- DUCKDB ANALİTİK RAPORLAMA METODU ---

  /// Kullanıcının girdiği SQL sorgusunu yerel veritabanında çalıştırır.
  /// DuckDB'nin güçlü JSON fonksiyonları ile `fields_json` alanındaki Firestore verilerini analiz edebilir.
  Future<List<Map<String, dynamic>>> runAnalyticalQuery(String sql) async {
    final stmt = _conn.prepare(sql);
    final res = stmt.execute();

    final List<Map<String, dynamic>> results = [];
    final columns = res.columnNames;

    for (final row in res) {
      final Map<String, dynamic> rowData = {};
      for (int i = 0; i < columns.length; i++) {
        rowData[columns[i]] = row[i];
      }
      results.add(rowData);
    }
    stmt.dispose();
    return results;
  }

  /// Veritabanı bağlantılarını düzgün şekilde kapatır.
  void dispose() {
    _conn.dispose();
    _db.dispose();
  }
}
```

---

## 3. API Veri Transfer Modelleri (Data Transfer Objects - DTO)

Firebase REST ve gRPC API'lerinden gelen verileri işlemek ve yerel modellere dönüştürmek için tasarlanmış temiz DTO yapıları aşağıda tanımlanmıştır.

### 3.1. FirestoreDocumentDto
Firestore REST/gRPC API'den dönen döküman verilerini sarmalar.

```dart
class FirestoreDocumentDto {
  final String id;
  final String path;
  final Map<String, dynamic> fields;
  final DateTime createTime;
  final DateTime updateTime;

  FirestoreDocumentDto({
    required this.id,
    required this.path,
    required this.fields,
    required this.createTime,
    required this.updateTime,
  });

  factory FirestoreDocumentDto.fromJson(Map<String, dynamic> json) {
    final fieldsMap = _parseFirestoreFields(json['fields'] ?? {});
    return FirestoreDocumentDto(
      id: (json['name'] as String).split('/').last,
      path: json['name'],
      fields: fieldsMap,
      createTime: DateTime.parse(json['createTime']),
      updateTime: DateTime.parse(json['updateTime']),
    );
  }

  Map<String, dynamic> toJson() {
    return {
      'fields': _convertToFirestoreFields(fields),
    };
  }

  static Map<String, dynamic> _parseFirestoreFields(Map<String, dynamic> fields) {
    final Map<String, dynamic> result = {};
    fields.forEach((key, value) {
      result[key] = _parseValue(value);
    });
    return result;
  }

  static dynamic _parseValue(Map<String, dynamic> val) {
    if (val.containsKey('stringValue')) return val['stringValue'];
    if (val.containsKey('integerValue')) return int.parse(val['integerValue']);
    if (val.containsKey('doubleValue')) return double.parse(val['doubleValue']);
    if (val.containsKey('booleanValue')) return val['booleanValue'];
    if (val.containsKey('nullValue')) return null;
    if (val.containsKey('timestampValue')) return DateTime.parse(val['timestampValue']);
    if (val.containsKey('geoPointValue')) {
      return {
        'latitude': val['geoPointValue']['latitude'],
        'longitude': val['geoPointValue']['longitude'],
      };
    }
    if (val.containsKey('mapValue')) {
      return _parseFirestoreFields(val['mapValue']['fields'] ?? {});
    }
    if (val.containsKey('arrayValue')) {
      final list = val['arrayValue']['values'] as List? ?? [];
      return list.map((item) => _parseValue(item as Map<String, dynamic>)).toList();
    }
    return null;
  }

  static Map<String, dynamic> _convertToFirestoreFields(Map<String, dynamic> map) {
    return {};
  }
}
```

### 3.2. AuthUserDto
Firebase Authentication API'sinden gelen kullanıcı verilerini temsil eder.

```dart
class AuthUserDto {
  final String uid;
  final String? email;
  final bool emailVerified;
  final String? displayName;
  final String? photoUrl;
  final String? phoneNumber;
  final bool disabled;
  final Map<String, dynamic> customClaims;
  final DateTime createdAt;
  final DateTime lastSignInAt;
  final List<String> providerIds;

  AuthUserDto({
    required this.uid,
    this.email,
    required this.emailVerified,
    this.displayName,
    this.photoUrl,
    this.phoneNumber,
    required this.disabled,
    required this.customClaims,
    required this.createdAt,
    required this.lastSignInAt,
    required this.providerIds,
  });

  factory AuthUserDto.fromJson(Map<String, dynamic> json) {
    return AuthUserDto(
      uid: json['localId'],
      email: json['email'],
      emailVerified: json['emailVerified'] ?? false,
      displayName: json['displayName'],
      photoUrl: json['photoUrl'],
      phoneNumber: json['phoneNumber'],
      disabled: json['disabled'] ?? false,
      customClaims: _parseCustomClaims(json['customAttributes']),
      createdAt: DateTime.fromMillisecondsSinceEpoch(int.parse(json['createdAt'])),
      lastSignInAt: DateTime.fromMillisecondsSinceEpoch(int.parse(json['lastLoginAt'])),
      providerIds: (json['providerUserInfo'] as List?)
              ?.map((item) => item['providerId'] as String)
              .toList() ?? [],
    );
  }

  static Map<String, dynamic> _parseCustomClaims(String? claimsStr) {
    if (claimsStr == null || claimsStr.isEmpty) return {};
    try {
      return {};
    } catch (_) {
      return {};
    }
  }
}
```

### 3.3. StorageNodeDto
Firebase Storage (Cloud Storage) API'sinden dönen klasör veya dosya bilgilerini tekilleştirir.

```dart
class StorageNodeDto {
  final String name;
  final String fullPath;
  final bool isFolder;
  final int? size;
  final String? contentType;
  final DateTime? updated;
  final String? downloadUrl;

  StorageNodeDto({
    required this.name,
    required this.fullPath,
    required this.isFolder,
    this.size,
    this.contentType,
    this.updated,
    this.downloadUrl,
  });

  factory StorageNodeDto.fromGoogleMetadata(Map<String, dynamic> json) {
    return StorageNodeDto(
      name: (json['name'] as String).split('/').last,
      fullPath: json['name'],
      isFolder: false,
      size: int.parse(json['size'] ?? '0'),
      contentType: json['contentType'],
      updated: DateTime.parse(json['updated']),
      downloadUrl: json['mediaLink'],
    );
  }

  factory StorageNodeDto.asFolder({required String name, required String fullPath}) {
    return StorageNodeDto(
      name: name,
      fullPath: fullPath,
      isFolder: true,
    );
  }
}
```
