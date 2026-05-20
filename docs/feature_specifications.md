# Fonksiyonel Özellik Detayları - Firebox

Bu doküman, Firebox uygulamasında yer alacak tüm ana modülleri, bu modüllerin alt özelliklerini, kullanıcı senaryolarını ve teknik gereksinimlerini ayrıntılı olarak açıklamaktadır.

---

## MODÜL 1: Workspace ve Bağlantı Yönetimi

Workspace modülü, kullanıcının farklı Firebase projelerine güvenli bir şekilde bağlanmasını, bu bağlantıları kaydetmesini ve aralarında hızlıca geçiş yapmasını sağlar.

### 1.1. Yeni Proje Bağlantısı Oluşturma
Kullanıcı uygulamaya yeni bir Firebase projesi eklemek istediğinde karşısına 4 farklı yöntem sunulur:
1. **Google Hesabı ile Giriş (OAuth 2.0)**:
   - Kullanıcı "Google ile Giriş Yap" butonuna tıklar.
   - Sistem varsayılan tarayıcıda Google yetkilendirme sayfasını açar.
   - Giriş başarılı olduğunda, kullanıcının erişim yetkisi olan tüm Firebase projeleri (GCP Projects) bir liste halinde çekilir.
   - Kullanıcı listeden istediği projeleri seçerek Workspace listesine ekler.
2. **Service Account JSON Yükleme**:
   - Kullanıcı Google Cloud Console'dan indirdiği `.json` uzantılı hizmet hesabı anahtar dosyasını sürükleyip bırakır veya dosya seçiciden seçer.
   - Firebox dosyayı doğrular (`project_id`, `private_key` ve `client_email` alanlarının varlığı kontrol edilir).
   - Bu yöntem, projeye tam "Firebase Admin" yetkisiyle erişim sağlar.
3. **Firebase CLI Entegrasyonu (Otomatik Keşif)**:
   - Uygulama, kullanıcının bilgisayarında daha önce Firebase CLI kullanıp kullanmadığını kontrol eder.
   - CLI'ın oluşturduğu yerel kimlik bilgileri dosyası (`~/.config/configstore/firebase-tools.json`) taranır.
   - Buradaki aktif oturumlar tespit edilerek projeler otomatik olarak içe aktarılır.
4. **Firebase Local Emulator Suite Bağlantısı**:
   - Geliştiricilerin yerel bilgisayarlarında çalışan emülatörlere bağlanmasını sağlar.
   - Kullanıcıdan Firestore, Auth ve Storage emülatörlerinin port bilgileri istenir (Örn: Firestore: 8080, Auth: 9099, Storage: 9199).
   - Emülatör bağlantıları için SSL/şifreleme aranmaz, yerel HTTP üzerinden hızlıca bağlanılır.

### 1.2. Workspace Özelleştirme ve Güvenlik Önlemleri
Bağlanan her proje bir "Workspace Card" haline gelir ve şu ayarlara sahip olur:
- **Proje Renklendirme (Project Coloring)**: Kullanıcı her projeye bir renk atayabilir (Örn: Production için Kırmızı, Staging için Turuncu, Development için Yeşil). Seçilen renk, uygulamanın başlık çubuğunda (Title Bar) ve yan menüsünde baskın olarak gösterilerek kullanıcının yanlış veritabanında işlem yapması (örn. canlı veriyi yanlışlıkla silmesi) engellenir.
- **Salt Okunur Modu (Read-Only Mode)**: Özellikle "Production" projeleri için kullanıcı bu modu aktif edebilir. Bu mod açıkken, arayüzdeki tüm ekleme, silme, güncelleme butonları pasif hale gelir. Kullanıcı veri yazmaya çalışırsa "Bu proje Salt Okunur modundadır" uyarısı alır.
- **Workspace Gruplama**: Çok sayıda projesi olan kullanıcılar için projeleri klasörleme/gruplama imkanı sunulur.

---

## MODÜL 2: Firestore Gezgini (Firestore Explorer)

Firestore veritabanındaki koleksiyonları, dökümanları ve alt koleksiyonları taramak, sorgulamak ve düzenlemek için geliştirilmiş en kapsamlı modüldür.

```
+------------------------------------------------------------------------------------+
|  [Search / Query Bar]  [+ Add Document]  [Saved Queries]  [Split View]  [ReadOnly] |
+------------------------------------------------------------------------------------+
|  COLLECTIONS        |  DOCUMENTS (Query Results)   |  DOCUMENT DETAILS (JSON/Tree) |
|  -----------------  |  --------------------------  |  ---------------------------- |
|  - users            |  - user_001                  |  {                            |
|  - orders           |  - user_002                  |    "name": "Ahmet",           |
|  - products         |  - user_003                  |    "email": "ahmet@mail.com", |
|    - sub_reviews    |  - user_004                  |    "location": Geopoint,      |
|                     |                              |    "roles": ["admin"]         |
|                     |                              |  }                            |
+------------------------------------------------------------------------------------+
```

### 2.1. Üçlü Görünüm Paneli (Table, Tree, JSON)
Kullanıcı döküman listesini ve detaylarını 3 farklı formatta görüntüleyebilir:
1. **Tablo Görünümü (Table View)**:
   - Excel benzeri bir tablo tasarımı sunulur. Sütunlar döküman alanlarını (fields), satırlar ise dökümanları temsil eder.
   - Sütun başlıklarına tıklanarak yerel sıralama yapılabilir. Sütunlar sürüklenebilir, gizlenebilir veya yeniden boyutlandırılabilir.
   - Hücrelere çift tıklanarak **Inline Düzenleme (Satır içi düzenleme)** yapılabilir. Değişiklikler anında Firestore'a kaydedilir.
2. **Ağaç Görünümü (Tree View)**:
   - İç içe geçmiş (nested) Map ve Array içeren dökümanlar için idealdir.
   - Karmaşık objeler daraltılabilir (collapse) ve genişletilebilir (expand).
   - Yeni bir alan eklemek için "+" butonuna tıklanarak veri tipi (String, Number, Boolean, Map, Array, Null, Timestamp, Geopoint, Document Reference) seçilir.
3. **JSON Görünümü (JSON View)**:
   - Dökümanı doğrudan saf bir JSON metni olarak görüntüler.
   - Entegre kod editörü sayesinde kullanıcı JSON üzerinde doğrudan değişiklik yapıp "Kaydet" dediğinde, JSON çözümlenir ve Firestore belgesi güncellenir. Sözdizimi (Syntax) hataları kaydetmeden önce doğrulanır.

### 2.2. Gelişmiş Sorgu Oluşturucu (Query Builder)
Firebase Console'un yetersiz kaldığı durumlarda Firebox gelişmiş sorgulama imkanı tanır:
- **Çoklu Filtreleme**: Kullanıcı birden fazla `where` koşulu ekleyebilir (Örn: `age >= 18` AND `status == 'active'`).
- **Farklı Alanlarda Sıralama**: Firestore kuralları gereği farklı alanlarda sıralama yapıldığında indeks gereklidir. Uygulama, eğer sorgu indeks eksikliğinden dolayı hata verirse, Firebase Console'un indeks oluşturma URL'ini içeren hatayı yakalar ve kullanıcıya "İndeks Oluştur" butonu sunar.
- **Özel Operatörler**: `starts-with` (başlangıç karakteri araması), `array-contains`, `array-contains-any`, `in`, `not-in` operatörleri arayüzden kolayca seçilebilir.
- **Sorgu Kaydetme (Saved Queries)**: Yazılan karmaşık sorgular isimlendirilerek kaydedilebilir. Böylece sonraki oturumlarda tek tıkla aynı sorgu çalıştırılır.

### 2.3. Coğrafi Veri (Geopoint) Görselleştirici ve Harita
- Eğer döküman içinde bir `Geopoint` (Enlem, Boylam) alanı varsa, dökümanın yanında bir "Haritada Göster" ikonu belirir.
- İkona tıklandığında, uygulama içinde entegre bir harita penceresi (OpenStreetMap tabanlı `flutter_map` kullanılarak) açılır ve koordinat harita üzerinde işaretlenir.
- Çoklu döküman seçildiğinde, tüm dökümanların konumları haritaya "Marker" olarak dökülür; marker'a tıklandığında ilgili döküman detayına gidilir.

### 2.4. Bölünmüş Ekran (Split View)
- Kullanıcı ekranı yatay veya dikey olarak ikiye bölebilir.
- Sol panelde "Development" veritabanının `users` koleksiyonunu, sağ panelde ise "Production" veritabanının `users` koleksiyonunu yan yana açabilir.
- Sol taraftan seçilen dökümanlar sürüklenip sağ tarafa bırakılarak (Drag and Drop) veya "Kopyala -> Yapıştır" yapılarak projeler arası döküman transferi gerçekleştirilebilir.

### 2.5. Gelişmiş CRUD İşlemleri
- **Döküman Taşıma ve Kopyalama**: Bir dökümanı başka bir koleksiyona veya tamamen başka bir Firebase projesine kopyalama/taşıma.
- **Döküman Yeniden Adlandırma (Rename)**: Firestore'da döküman kimliği (ID) doğrudan değiştirilemez. Firebox bunu kolaylaştırmak için arka planda hedef ID ile dökümanın birebir kopyasını oluşturur, başarılı olursa eski dökümanı siler (Transaction içinde güvenli işlem).
- **Alt Koleksiyon (Subcollection) Yönetimi**: Dökümanın altındaki alt koleksiyonlar hiyerarşik ağaç yapısında listelenir, yeni alt koleksiyonlar oluşturulabilir.

---

## MODÜL 3: Firebase Authentication Yönetici Paneli

Firebase Auth üzerindeki kullanıcı kayıtlarını yönetmek için geliştirilmiş güçlü bir admin arayüzüdür.

### 3.1. Kullanıcı Listeleme, Arama ve Sıralama
- Kullanıcılar e-posta, telefon numarası veya UID değerine göre anlık olarak aranabilir.
- Listede kullanıcının profil resmi (varsa), e-postası, sağlayıcısı (Google, Apple, Password vb.), oluşturulma tarihi ve son giriş saati dakika hassasiyetiyle gösterilir.
- Kullanıcılar kayıt tarihine veya son oturum açma tarihine göre artan/azalan şekilde sıralanabilir.

### 3.2. Kullanıcı Bilgisi Güncelleme ve CRUD
- **Şifre ve E-posta Değiştirme**: Kullanıcı şifresi doğrudan admin yetkisiyle sıfırlanabilir veya değiştirilebilir. E-posta adresi güncellenebilir ve "E-posta doğrulandı mı?" (emailVerified) durumu `true`/`false` olarak değiştirilebilir.
- **Kullanıcı Hesabı Dondurma/Silme**: Tek tıkla kullanıcı hesabı dondurulabilir (disable) veya tamamen silinebilir.
- **Bulk (Toplu) İşlemler**: Arayüzden çoklu seçim yapılarak yüzlerce kullanıcı tek seferde dondurulabilir veya silinebilir.

### 3.3. Custom Claims (Özel Yetkiler) Editörü
Firebase Auth'un en güçlü özelliklerinden biri olan custom claims, kullanıcı rollerini belirler. Firebox içinde bu işlem için özel bir görsel editör yer alır:
- Kullanıcının detay panelinde "Custom Claims" sekmesi bulunur.
- Kullanıcı buraya `{"admin": true, "premium": false, "department": "sales"}` gibi JSON verileri girer.
- Firebox bunu doğrular ve Firebase Auth Admin SDK API'si üzerinden kullanıcının token'ına yazar. Kullanıcının bir sonraki oturum açışında bu yetkiler geçerli olur.

### 3.4. Auth Verisi Aktarma (Import/Export)
- **Dışa Aktarma (Export)**: Tüm auth kullanıcıları şifre hash'leri de dahil olmak üzere (veya hash'siz olarak) CSV ya da JSON formatında bilgisayara indirilebilir.
- **İçe Aktarma (Import)**: CSV veya JSON formatındaki kullanıcı listesi sisteme yüklenir. Şifre hash algoritması (bcrypt, scrypt, md5 vb.) seçilerek kullanıcıların şifreleriyle birlikte aktarılması sağlanır.

---

## MODÜL 4: Firebase Storage Yöneticisi (Yeni Özellik)

Uygulamanın Cloud Storage üzerindeki dosyaları tıpkı bir işletim sistemi dosya gezgini gibi yönetmesini sağlayan kritik modüldür.

```
+------------------------------------------------------------------------------------+
|  [Path: gs://my-app.appspot.com/users/avatars/]           [+ Upload File] [+ Folder] |
+------------------------------------------------------------------------------------+
|  [x] Name                 |  Size       |  Type         |  Last Modified           |
|  --------------------------------------------------------------------------------- |
|  [D] .. (Parent Folder)   |  -          |  Folder       |  -                       |
|  [F] avatar_01.png        |  145 KB     |  image/png    |  2026-05-19 14:30        |
|  [F] tutorial.mp4         |  12.4 MB    |  video/mp4    |  2026-05-18 09:12        |
|  [F] config.json          |  1.2 KB     |  text/json    |  2026-05-17 11:45        |
+------------------------------------------------------------------------------------+
|  [QUEUE MANAGER]  avatar_02.png (85% - 1.2 MB/s) | tutorial_v2.mp4 (Sırada)       |
+------------------------------------------------------------------------------------+
```

### 4.1. Klasör ve Dosya Gezgini
- **Ağaç ve Tablo Düzeni**: Dosyalar hiyerarşik klasör yapısında taranır. Boyut, dosya türü ve son değiştirilme tarihlerine göre listelenir.
- **Sürükle-Bırak Yükleme**: Kullanıcı bilgisayarından sürüklediği bir veya birden fazla dosya/klasörü doğrudan Firebox penceresine bırakarak Storage'a yükleyebilir.
- **Klasör Oluşturma**: Firebase Storage'da gerçek klasörler bulunmaz (nesne anahtarı yapısı - object key path). Uygulama boş klasör yapısını simüle etmek için arkada `.placeholder` dosyası oluşturarak kullanıcıya gerçekçi bir klasör oluşturma deneyimi sunar.

### 4.2. İndirme ve Yükleme Kuyruğu (Queue Manager)
Büyük boyutlu dosya transferleri için ekranın alt kısmında veya ayrı bir sekmede çalışan bir kuyruk yöneticisi bulunur:
- **Eşzamanlılık Kontrolü**: Aynı anda en fazla 3 transfer aktif olarak çalışır, diğerleri sıraya alınır.
- **Kontrol Butonları**: Her transfer için "Duraklat" (Pause), "Devam Et" (Resume) ve "İptal Et" (Cancel) butonları sunulur.
- **Hız ve Süre Gösterimi**: Transfer hızı (MB/s) ve kalan tahmini süre (ETA) dinamik olarak hesaplanarak gösterilir.

### 4.3. Entegre Medya Önizleyicileri (In-App Previews)
Kullanıcı dosyayı indirmeden önce doğrudan Firebox içerisinde önizleyebilir:
- **Resim Önizleyici**: PNG, JPG, WEBP, GIF formatındaki resimler için yakınlaştırma/uzaklaştırma destekli görsel önizleme alanı.
- **Video Oynatıcı**: MP4, WebM formatındaki videolar için entegre oynatıcı (duraklat, sar, ses ayarla).
- **Ses Oynatıcı**: MP3, WAV formatındaki ses dosyaları için ses dalga şekli (waveform) ve oynatıcı barı.
- **Metin ve Kod Editörü**: `.txt`, `.json`, `.js`, `.yaml`, `.xml` gibi dosyalar indirilmeden önce uygulama içi editörde açılır, düzenlenip doğrudan Storage üzerine kaydedilebilir.

### 4.4. Dosya Metadata Düzenleme
Dosyaya sağ tıklanıp "Özellikler" denildiğinde şu alanlar düzenlenebilir:
- `Content-Type` (Dosya MIME türü).
- `Cache-Control` (Tarayıcı önbellekleme süresi).
- `Custom Metadata`: Kullanıcının kendi tanımladığı anahtar-değer (Key-Value) çiftleri (Örn: `owner: user_99`, `approved: true`).

---

## MODÜL 5: JavaScript Scripting Shell

Geliştiricilerin Firestore ve Auth üzerinde toplu veri analizleri ve şema değişiklikleri yapabilmesi için tasarlanmış kodlama alanıdır.

### 5.1. Kod Editörü Özellikleri
- Monaco Editor veya optimize edilmiş bir Flutter kod editörü (`code_text_field`) entegrasyonu.
- Firebase Javascript Admin SDK syntax yapısına uygun otomatik tamamlama (Autocomplete) ve renklendirme.
- Sık kullanılan betik şablonları (Templates): "Tüm dökümanlarda alan adı değiştirme", "Belirli koşuldaki kullanıcıları toplu silme", "Koleksiyon yedekleme".

### 5.2. Konsol ve Çıktı Paneli
- Betik çalıştırıldığında alt kısımda bir çıktı terminali açılır.
- `console.log()` ve `console.error()` çıktıları anlık olarak satır numaralarıyla buraya yazılır.
- Betiğin toplam çalışma süresi, etkilenen döküman sayısı ve harcanan read/write işlem adetleri özet rapor olarak gösterilir.
- Kullanıcı dilerse çalışan betiği yarıda kesmek için "Durdur" (Kill Process) butonunu kullanabilir.

---

## MODÜL 6: Veri Taşıma ve Göç Araçları (Import/Export & Migration)

Projeler arası veri geçişini ve yedeklemeyi kolaylaştıran sihirbaz modülüdür.

### 6.1. Firestore Veri Aktarım Sihirbazı (Import Wizard)
Kullanıcı bir CSV veya JSON dosyasını Firestore koleksiyonuna aktarmak istediğinde şu adımları izler:
1. **Dosya Seçimi**: Dosya yüklenir ve ilk 5 satır önizleme olarak gösterilir.
2. **Alan Eşleme (Field Mapping)**: CSV kolonları ile Firestore alan adları eşleştirilir.
3. **Veri Tipi Belirleme**: Uygulama kolonları analiz ederek veri tiplerini (String, Number, Boolean, Timestamp) otomatik tahmin eder; kullanıcı dilerse manuel değiştirebilir.
4. **Döküman Kimliği (Doc ID) Ayarı**: Döküman ID'lerinin otomatik (auto-generated ID) mi oluşturulacağı yoksa CSV'deki belirli bir kolondan (örn. `user_id` kolonu) mu alınacağı seçilir.
5. **Yürütme**: Aktarım başlatılır, hata veren satırlar ayrı bir log dosyası olarak bilgisayara indirilebilir.

### 6.2. Canlı Proje Transferi (Migration Wizard)
İki farklı Workspace arasında (Örn: Local Emulator'den Production'a veya Dev'den Staging'e) veri transferi sağlar:
- Kullanıcı kaynak projeyi, hedef projeyi ve taşınacak koleksiyonları seçer.
- İsterse sadece belirli sorgu koşullarına uyan dökümanların taşınmasını sağlayabilir.
- Transfer işlemi arka planda dökümanları paketler halinde (batches of 500) okuyup hedef projeye yazarak gerçekleştirilir. İlerleme çubuğunda yüzde ve transfer hızı gösterilir.
