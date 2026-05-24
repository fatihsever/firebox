# UI/UX Tasarım ve Arayüz Rehberi - Firedock

Bu doküman, Firedock uygulamasının responsive (duyarlı) ekran yerleşim tasarımlarını, görsel kimliğini, renk sistemini, tipografisini ve kullanıcı etkileşim (etkileşim) standartlarını belirler.

---

## 1. Tasarım Felsefesi ve Görsel Dil

Firedock, yoğun veri içeren ve uzun saatler boyunca kullanılan teknik bir yönetim arayüzüdür. Bu nedenle tasarım dilinin temel amacı **göz yorgunluğunu azaltmak**, **hızlı navigasyon sağlamak** ve **operasyonel hataları en aza indirmektir**.

- **Birincil Öncelik: Dark-First (Koyu Tema Odaklı)**: Geliştiricilerin gece ve loş ışıkta çalışma alışkanlıkları göz önünde bulundurularak uygulama öncelikli olarak koyu tema odaklı tasarlanır. Açık tema (Light Mode) alternatif bir seçenek olarak sunulur.
- **SaaS Dashboard Stili**: Temiz sınırlar, keskin hatlar, minimal gölgeler ve düz renk blokları içeren modern bir endüstriyel arayüz stili benimsenmiştir.
- **Odaklanmış Dikkat**: Kritik eylemler (Örn: Silme, veri üzerine yazma) dışında parlak ve doygun renklerden kaçınılır. Arayüzün geneli nötr tonlardadır.

---

## 2. Renk Sistemi (Color Palette)

Arayüzde kullanılan renkler, Firebase'in klasik sarı-turuncu paletinden ziyade, modern bir yazılım stüdyosu hissi veren indigo ve derin slate tonlarına dayanır.

### 2.1. Temel Renkler (Global Palette)
| Renk Rolü | Hex Kodu | Görsel Karşılığı / Kullanım Alanı |
| :--- | :--- | :--- |
| **Primary (Birincil)** | `#6366F1` | Indigo-500: Butonlar, aktif sekmeler, vurgulanmış alanlar. |
| **Secondary (İkincil)**| `#3B82F6` | Blue-500: Bilgilendirme etiketleri, ikincil aksiyonlar. |
| **Dark Background**   | `#0F172A` | Slate-900: Ana ekran arka planı (Koyu mod). |
| **Dark Surface**      | `#1E293B` | Slate-800: Kartlar, sol menü, detay panelleri arka planı. |
| **Dark Border**       | `#334155` | Slate-700: Panel ayırıcı çizgiler, input kenarlıkları. |
| **Text Primary**      | `#F8FAFC` | Slate-50: Okunabilirliği en yüksek ana metinler. |
| **Text Secondary**    | `#94A3B8` | Slate-400: Tarihler, yardımcı yazılar, pasif alanlar. |
| **Success (Başarı)**  | `#10B981` | Emerald-500: Tamamlanan transferler, başarılı işlemler. |
| **Warning (Uyarı)**   | `#F59E0B` | Amber-500: Geri alınamaz işlemler öncesi uyarılar. |
| **Danger (Tehlike)**  | `#EF4444` | Red-500: Silme butonları, hatalı bağlantılar, prod ortam uyarıları. |

### 2.2. Proje Renklendirme Sistemi (Workspace Badges)
Kullanıcının projelerini görsel olarak birbirinden ayırabilmesi için sol menüde ve üst barda seçilen proje rengi baskın olur:
- `#EF4444` (Kırmızı): **Production** (Hata riskini azaltmak için kırmızı çerçeve ve uyarı barları aktif olur).
- `#F59E0B` (Turuncu): **Staging / Test**.
- `#10B981` (Yeşil): **Development**.
- `#3B82F6` (Mavi): **Local Emulator**.

---

## 3. Responsive Ekran Düzeni (Layout Grid)

Firedock, hem geniş masaüstü ekranlarında (Mac/Windows/Linux) hem de dar mobil ekranlarda (iOS/Android) kusursuz çalışmalıdır.

### 3.1. Masaüstü Düzeni (Desktop Layout) - Min: 1024px
Masaüstü ekranlar **Üç Sütunlu Esnek Izgara** yapısını kullanır:

```
+-----------------------------------------------------------------------------------+
|  Title Bar (Workspace Selector, Project Indicator, Search Bar, Settings)          |
+-----------------------------------------------------------------------------------+
|  Sidebar (15%)  |  Navigation Tree (25%)  |  Main Working Area (60%)              |
|                 |                         |                                       |
|  - Firestore    |  - users                |  +---------------------------------+  |
|  - Auth         |  - orders               |  |  Table / Tree / JSON View       |  |
|  - Storage      |  - products             |  |                                 |  |
|  - Scripting    |    - categories         |  |                                 |  |
|  - Migration    |                         |  |                                 |  |
|                 |                         |  +---------------------------------+  |
|                 |                         |  |  Details Panel (Split View)     |  |
+-----------------+-------------------------+---------------------------------------+
```

1. **Sidebar (Sol Menü - %15 Genişlik)**: Firestore, Auth, Storage, Scripting, Migration modülleri arasında geçişi sağlar. En altta yerel ayarlar butonu bulunur.
2. **Navigation Tree (Gezinme Ağacı - %25 Genişlik)**: Firestore için koleksiyon listesi; Storage için klasör yapısı hiyerarşik bir ağaç şeklinde bu sütunda listelenir.
3. **Main Working Area (Ana Çalışma Alanı - %60 Genişlik)**: Sorgu sonuçları, tablolar veya kod editörü bu alanda tam ekran veya yatay bölünmüş (Split View) olarak yer alır.

### 3.2. Mobil Düzen (Mobile Layout) - Max: 600px
Mobil ekranlarda yer kısıtlı olduğundan tasarım **Akışkan Tek Sütun** yapısına evrilir:
- **Bottom Navigation Bar**: Ana modüller (Firestore, Auth, Storage, Settings) arasında geçiş için alt bar kullanılır.
- **Kaydırılabilir Paneller (Swipe Navigation)**: Koleksiyon listesinden döküman listesine geçiş sağa/sola kaydırarak veya ekran içi kartlara tıklayarak sağlanır.
- **Detay Panelleri için Bottom Sheet**: Bir Firestore dökümanının detayı veya Storage dosyasının özellikleri ekranın altından yukarı doğru açılan yarım pencereler (Bottom Sheets) ile gösterilir. Bu sayede tek elle kullanım kolaylaşır.

---

## 4. Kullanıcı Etkileşimi Standartları (Interaction)

### 4.1. Komut Paleti (Command Palette) - `Cmd + K` / `Ctrl + K`
- Kullanıcı hangi ekranda olursa olsun bu kısayola bastığında ekranın ortasında yarı saydam, odaklanmış bir arama kutusu açılır.
- Kullanıcı buraya yazarak şunları yapabilir:
  - Kaydedilmiş sorguları bulup çalıştırma (Örn: `> run query: Premium Users`).
  - Projeler arası hızlı geçiş yapma (Örn: `> switch: Dev Environment`).
  - Doğrudan bir koleksiyona gitme (Örn: `> go: orders`).
  - Uygulama ayarlarını değiştirme (Örn: `> toggle: dark mode`).

### 4.2. Sürükle-Bırak (Drag and Drop)
- **Masaüstünde**: Dosya yüklemek için Storage ekranına dışarıdan dosya bırakılması. Split View açıkken bir dökümanın sol tablodan sağ tabloya sürüklenerek kopyalanması.
- **Mobilde**: Döküman listesinde satırı sağa kaydırarak (Swipe Right) hızlı kopyalama, sola kaydırarak (Swipe Left) hızlı silme aksiyonları.

### 4.3. Klavye Kısayolları (Keyboard Shortcuts)
- `Esc`: Açık olan modal veya detay panelini kapatır.
- `Cmd + S` / `Ctrl + S`: JSON editöründeki değişiklikleri veya kod yazma alanındaki betiği kaydeder.
- `Cmd + Enter` / `Ctrl + Enter`: Yazılan sorguyu veya JS betiğini çalıştırır.
- `Tab` / `Shift + Tab`: Tablo görünümünde hücreler arası yatay gezinme sağlar.
