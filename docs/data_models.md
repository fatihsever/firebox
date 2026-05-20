# Veri Modelleri ve Yerel Depolama - Firebox

Bu doküman, Firebox uygulamasının yerel veritabanında (Isar DB) tutacağı verilerin şemalarını, uygulama içi kullanılan veri modellerini ve Firebase API'leri ile haberleşirken kullanılan DTO (Data Transfer Object) yapılarını tanımlamaktadır.

---

## 1. Yerel Veritabanı Şemaları (Isar Database Schemas)

Yerel veritabanı, kullanıcının uygulama ayarlarını, bağlantı profillerini ve geçmiş işlemlerini saklamak için kullanılır. Aşağıda Dart sınıfları ve Isar anotasyonları ile tanımlanmış şemalar yer almaktadır.

### 1.1. Workspace (Çalışma Alanı / Proje Bağlantısı)
Kullanıcının kaydettiği her bir Firebase projesini temsil eder. Hassas bilgiler şifrelenmiş olarak `flutter_secure_storage` içerisinde saklanır, bu modelde sadece referanslar ve konfigürasyonlar tutulur.

```dart
import 'package:isar/isar.dart';

part 'workspace.g.dart';

@collection
class WorkspaceModel {
  Id id = Isar.autoIncrement;

  @Index(unique: true)
  late String projectId; // Firebase Proje Kimliği (Örn: firebox-prod-123)

  late String name; // Kullanıcının verdiği ad (Örn: Canlı Sunucu)
  
  late String colorHex; // UI'da kullanılacak proje rengi (Örn: #EF4444)

  late bool isReadOnly; // Yanlışlıkla silmeleri önlemek için salt okunur modu

  @enumerated
  late AuthMethod authMethod; // googleSignIn, serviceAccount, emulator, cli

  String? serviceAccountFileName; // Servis hesabı dosya adı (Görsel bilgi için)

  // Emülatör portları (Eğer authMethod == emulator ise kullanılır)
  int? firestoreEmulatorPort;
  int? authEmulatorPort;
  int? storageEmulatorPort;

  late DateTime createdAt;
  late DateTime lastAccessedAt;
}

enum AuthMethod {
  googleSignIn,
  serviceAccount,
  emulator,
  cli,
}
```

### 1.2. SavedQuery (Kaydedilen Sorgular)
Firestore Explorer üzerinde kullanıcının tekrar tekrar kullanmak için kaydettiği sorgulardır.

```dart
import 'package:isar/isar.dart';

part 'saved_query.g.dart';

@collection
class SavedQueryModel {
  Id id = Isar.autoIncrement;

  late int workspaceId; // Hangi projeye ait olduğu bilgisi

  late String name; // Sorguya verilen ad (Örn: Aktif Premium Üyeler)

  late String collectionPath; // Koleksiyon yolu (Örn: users veya orders/123/items)

  late String filtersJson; // Filtre dizisinin JSON hali (Örn: [{"field":"age","op":">=","value":18}])

  String? orderByField; // Sıralama yapılacak alan adı

  late bool orderByDescending; // Azalan sıralama mı?

  int? limit; // Çekilecek maksimum döküman sayısı

  late DateTime createdAt;
}
```

### 1.3. TransferHistory (Veri Taşıma Geçmişi)
Kullanıcının projeler arası yaptığı transferlerin veya import/export işlemlerinin kaydıdır.

```dart
import 'package:isar/isar.dart';

part 'transfer_history.g.dart';

@collection
class TransferHistoryModel {
  Id id = Isar.autoIncrement;

  late int sourceWorkspaceId;
  int? targetWorkspaceId; // Dosya dışa aktarımında boş olabilir

  @enumerated
  late TransferType type; // firestore, auth, storage

  @enumerated
  late TransferStatus status; // running, completed, failed, paused

  late int processedCount; // Taşınan döküman/dosya sayısı
  late int totalCount; // Toplam döküman/dosya sayısı

  String? errorMessage; // Varsa hata detayları
  
  late DateTime startedAt;
  DateTime? endedAt;
}

enum TransferType { firestore, auth, storage }
enum TransferStatus { running, completed, failed, paused }
```

### 1.4. AppSettings (Uygulama Genel Ayarları)
Uygulamanın genel davranışını ve görünümünü belirleyen ayarlardır.

```dart
import 'package:isar/isar.dart';

part 'app_settings.g.dart';

@collection
class AppSettingsModel {
  Id id = Isar.autoIncrement;

  late String themeMode; // 'light', 'dark', 'system'
  
  late String languageCode; // 'tr', 'en'

  late String timestampFormat; // 'yyyy-MM-dd HH:mm:ss' vb.

  late String defaultMapProvider; // 'openstreetmap', 'mapbox'

  late bool enableAutocomplete; // JS kod editöründe otomatik tamamlama açık mı?
}
```

---

## 2. API Veri Transfer Modelleri (Data Transfer Objects - DTO)

Firebase API'lerinden dönen ham verileri Flutter içinde işlemek için kullanılan hafif ve performanslı veri modelleridir.

### 2.1. FirestoreDocumentDto
Firestore REST API'den dönen döküman verilerini sarmalar.

```dart
class FirestoreDocumentDto {
  final String id; // Döküman ID'si
  final String path; // Tam döküman yolu (Örn: projects/p1/databases/(default)/documents/users/u1)
  final Map<String, dynamic> fields; // Tip dönüştürmesi yapılmış alanlar
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
    // Google API formatındaki ham JSON'ı Dart map yapısına dönüştüren lojik
    // Örn: {"name": {"stringValue": "Ahmet"}} -> {"name": "Ahmet"}
    final fieldsMap = _parseFirestoreFields(json['fields'] ?? {});
    return FirestoreDocumentDto(
      id: json['name'].split('/').last,
      path: json['name'],
      fields: fieldsMap,
      createTime: DateTime.parse(json['createTime']),
      updateTime: DateTime.parse(json['updateTime']),
    );
  }

  Map<String, dynamic> toJson() {
    // Dart map yapısını Google API Firestore formatına geri dönüştüren lojik
    return {
      'fields': _convertToFirestoreFields(fields),
    };
  }
}
```

### 2.2. AuthUserDto
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
}
```

### 2.3. StorageNodeDto
Firebase Storage (Cloud Storage) API'sinden dönen klasör veya dosya bilgilerini tekilleştirir.

```dart
class StorageNodeDto {
  final String name; // Dosya veya klasör adı
  final String fullPath; // Bucket içindeki tam yol (Örn: users/avatars/avatar_01.png)
  final bool isFolder; // Nesne bir klasör mü yoksa dosya mı?
  final int? size; // Dosya boyutu (Klasörler için boştur)
  final String? contentType; // MIME tipi (Örn: image/png)
  final DateTime? updated; // Son güncellenme tarihi
  final String? downloadUrl; // Doğrudan erişim linki

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
      name: json['name'].split('/').last,
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
