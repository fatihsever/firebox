# Firedock - Flutter Tabanlı Firebase Yönetim ve Geliştirme Platformu

**Firedock**, Firebase projelerinizi en üst düzey verimlilikle yönetmeniz için tasarlanmış, **Flutter** tabanlı, çapraz platform (Masaüstü ve Mobil) destekli yeni nesil bir GUI (Grafik Kullanıcı Arayüzü) istemcisidir.

Firefoo'nun masaüstünde sunduğu gelişmiş Firestore ve Authentication özelliklerini temel alan Firedock, bu özellikleri modern bir mobil deneyim, güçlü bir **Firebase Storage Yönetim Modülü**, yerel analitik veritabanı **DuckDB** ve çapraz platform (Masaüstü & Mobil) destekli gelişmiş bir **Ödeme/Lisanslama Altyapısı** ile taçlandırmaktadır. Tamamen yerel (local-first) ve güvenli çalışan Firedock, verilerinizin doğrudan sizin makineniz ile Firebase sunucuları arasında akmasını sağlar; aracı bir sunucu (proxy) barındırmaz.

---

## 🗺️ Dokümantasyon Haritası ve İndeksi

Bu dizinde, Firedock projesinin sıfırdan son ürüne kadar geliştirilmesi için gerekli olan tüm teknik, mimari, veri ve tasarım dokümantasyonu yer almaktadır. Dokümanlar, bir yazılım geliştirme ekibinin başka hiçbir kaynağa ihtiyaç duymadan uygulamayı uçtan uca geliştirebilmesi için detaylandırılmıştır.

| Doküman | İçerik ve Amacı | Hedef Okuyucu |
| :--- | :--- | :--- |
| [Sistem Mimarisi](system_architecture.md) | **Feature-First Clean Architecture** uygulama yapısı, teknoloji yığını, state management (Riverpod), `dart_duckdb` entegrasyonu, JavaScript betik çalıştırma sanal makinesi (VM), platform entegrasyonları ve güvenlik mimarisi. | Sistem Mimarları, Kıdemli Geliştiriciler, Güvenlik Uzmanları |
| [Özellik Detayları](feature_specifications.md) | Workspace, Firestore Explorer, Auth Admin, Storage Manager, JS Scripting Shell ve Veri Aktarım modüllerinin detaylı fonksiyonel analizleri. | Flutter Geliştiricileri, İş Analistleri, Ürün Yöneticileri |
| [Veri Modelleri ve Yerel Depolama](data_models.md) | **DuckDB** SQL ilişkisel şemaları (Workspaces, Saved Queries, Transfer History, App Settings), Firestore/Auth/Storage API'leri için kullanılacak veri modelleri (DTO) ve Dart DuckDB servis kod örnekleri. | Veritabanı Mimarları, Flutter Geliştiricileri |
| [Lisanslama ve Ödeme Altyapısı](licensing_and_monetization.md) | Aylık/Yıllık/Ömür Boyu abonelikler, manuel lisans anahtarı doğrulamalı çapraz platform lisans senkronizasyonu (Stripe/Paddle & App Store/Google Play), Kurumsal toplu lisanslama altyapısı ve DuckDB lisans tabloları. | İş Geliştiriciler, DevOps Uzmanları, Flutter Geliştiricileri |
| [UI/UX Tasarım Rehberi](ui_ux_guidelines.md) | Responsive ekran düzenleri (Split View, Multi-Window), Dark Mode odaklı renk paleti, tipografi, özel proje renklendirme sistemi ve Komut Paleti tasarımı. | UI/UX Tasarımcıları, Frontend Geliştiricileri |
| [Geliştirme Yol Haritası](development_roadmap.md) | Kullanılacak paketlerin (pub.dev) tam listesi, 6 ana faza bölünmüş 18 haftalık proje planı, test stratejileri ve platform bazlı (OS/Mobile) dağıtım süreçleri. | Proje Yöneticileri, Scrum Master'lar, DevOps Uzmanları |

---

## 🎯 Temel Vizyon ve Fark Yaratan Özellikler

1. **Çoklu Platform Gücü (Flutter Desktop & Mobile)**: Mac, Windows, Linux, iOS ve Android üzerinde tamamen aynı iş mantığıyla, ancak platforma özel optimize edilmiş arayüzlerle çalışır.
2. **Gelişmiş Firestore Gezgini**: Verilerinizi sadece basit listeler halinde değil; Excel benzeri bir **Tablo**, hiyerarşik bir **Ağaç** veya doğrudan ham **JSON** olarak inceleyip düzenleyin.
3. **Yerel Analitik Gücü (DuckDB Integration)**: Firedock, standart NoSQL araçlarının aksine yerel veritabanı olarak **DuckDB** kullanır. Bu sayede, Firebase verilerinizin yerel yedekleri üzerinde karmaşık SQL sorguları, agregasyonlar, gelişmiş raporlamalar ve yerel yapay zeka (AI) analizleri koşturabilirsiniz.
4. **JS Scripting Shell**: SQL dünyasındaki veritabanı yönetim araçlarında olduğu gibi, Firestore veritabanınız üzerinde toplu güncellemeleri, şema değişikliklerini ve veri analizlerini JavaScript yazarak saniyeler içinde gerçekleştirin.
5. **Bütünleşik Firebase Storage Yöneticisi**: Firefoo'da bulunmayan, Firebase projelerinin olmazsa olmazı Cloud Storage için sürükle-bırak destekli, entegre medya oynatıcılu ve yükleme/indirme kuyruk yöneticili tam donanımlı bir arayüz.
6. **Güvenlik Odaklı Mimari (Secure-by-Design)**: Firebase Service Account anahtarlarınız veya Google OAuth oturum bilgileriniz asla üçüncü taraf sunuculara gitmez; cihazınızın işletim sistemine özel güvenli kasasında (Keychain/Keystore) şifreli olarak saklanır.
7. **Çapraz Ödeme & Portatif Lisanslama**: Masaüstünden (Stripe/Paddle) satın alınan bir lisans mobil cihazlarda, mobilden (Apple IAP / Google Billing) satın alınan abonelik ise masaüstünde anında geçerli olur. Kurumsal firmalar için kolay, manuel toplu lisanslama seçeneği içerir.
8. **Bölünmüş Ekran (Split View)**: İki farklı koleksiyonu veya aynı koleksiyonun iki farklı sorgu sonucunu yan yana açarak verileri görsel olarak karşılaştırın ve sürükleyerek taşıyın.

---

## 🛠️ Hızlı Başlangıç Geliştirici Kılavuzu

Projenin geliştirilmesine başlamak için aşağıdaki adımları takip edin:

1. **Ortamı Hazırlayın**: Flutter SDK'nın en güncel kararlı (stable) sürümünün sisteminizde kurulu olduğundan emin olun (`flutter doctor`).
2. **Mimariyi İnceleyin**: Geliştirmeye başlamadan önce [Sistem Mimarisi](system_architecture.md) dokümanını okuyarak **Feature-First Clean Architecture** (Özellik Odaklı Temiz Mimari) yapısını ve Riverpod durum yönetimi modelini kavrayın.
3. **Veritabanını ve Veri Modellerini Oluşturun**: [Veri Modelleri ve Yerel Depolama](data_models.md) dosyasındaki DuckDB SQL tablolarını kurun ve `dart_duckdb` bağlantı servislerini inceleyin.
4. **Lisanslama Altyapısını Öğrenin**: [Lisanslama ve Ödeme Altyapısı](licensing_and_monetization.md) dokümanını okuyarak Stripe/Adapty entegrasyon kurallarını ve çapraz platform lisanslama akışını anlayın.
5. **UI Geliştirmeye Başlayın**: [UI/UX Tasarım Rehberi](ui_ux_guidelines.md) doğrultusunda, masaüstü için split-view destekli geniş ekran düzenini kurun.
6. **Fazları Takip Edin**: Proje takvimini ve sprint dağılımlarını görmek için [Geliştirme Yol Haritası](development_roadmap.md) dokümanını referans alın.
