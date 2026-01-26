# Barkod ve QR Kod Türleri: Teknik Kılavuz

## Giriş

Barkod ve QR kod (Karekod) dünyası, temelde veriyi makinenin okuyabileceği görsel bir formatta saklama sanatıdır. Bir yazılımcı olarak bu türleri seçerken dikkate almanız gereken üç ana faktör vardır: Veri kapasitesi, okunabilirlik (dayanıklılık) ve sektörel standartlar. Bu makalede, en yaygın kullanılan barkod türlerini ve teknik detaylarını inceleyeceğiz.

## 1D (Çizgisel) Barkodlar

Bunlar klasik, dikey çizgilerden oluşan barkodlardır. Genellikle sadece sayısal veya sınırlı alfanümerik veri tutarlar. Veri tabanındaki bir kayda (ID) referans vermek için kullanılırlar; verinin kendisini taşımazlar.

### EAN-13 ve UPC-A (Perakende Standartları)

Marketlerdeki ürünlerin üzerinde gördüğünüz standart barkodlardır.

- **Kullanım Alanı**: Perakende, POS (Satış Noktası) sistemleri.
- **Veri Tipi**: Sadece sayısal.
- **Kapasite**: 13 hane (EAN) veya 12 hane (UPC).
- **Özellik**: Küresel olarak benzersizdir (GS1 standartlarına tabidir). Türkiye'deki ürünler genellikle 869 ile başlar.

### Code 128 (Lojistiğin Kralı)

Yazılımcılar için en esnek 1D barkoddur.

- **Kullanım Alanı**: Kargo etiketleri, stok takibi, dahili envanter yönetimi.
- **Veri Tipi**: Tüm ASCII karakterler (Harf, sayı, sembol).
- **Kapasite**: Değişkendir ancak çok uzarsa okuması zorlaşır.
- **Özellik**: Yüksek yoğunlukludur (az yere çok veri sığdırır). Kendi projelerinizde (örneğin personel kartı veya demirbaş takibi) kullanmak için en güvenli 1D seçimdir.

### Code 39 (Eski Toprak)

- **Kullanım Alanı**: Otomotiv, savunma sanayi, kimlik kartları.
- **Veri Tipi**: Büyük harfler (A-Z), rakamlar (0-9) ve birkaç özel karakter (- . $ / + %).
- **Dezavantajı**: Code 128'e göre çok daha fazla yer kaplar (düşük yoğunluk).

## 2D (Matris) Kodlar

Veriyi hem yatay hem dikey olarak saklarlar. Bu sayede 1D barkodlara göre yüzlerce kat daha fazla veri taşıyabilirler. Verinin kendisini (örneğin bir URL, vCard veya JSON parçası) taşıyabilirler.

### QR Code (Quick Response)

En popüler 2D koddur. 3 köşesindeki kareler (finder patterns) sayesinde çok hızlı ve her açıdan (360 derece) okunabilir.

- **Kullanım Alanı**: URL yönlendirme, ödeme sistemleri, WiFi paylaşımı, basit metin aktarımı.
- **Kapasite**: Sayısal olarak ~7000, alfanümerik olarak ~4000 karaktere kadar.
- **Hata Düzeltme (Error Correction)**: En büyük gücüdür. Kodun %30'u kirlense veya yırtılsa bile okunabilir (Seviye H seçilirse).

### Data Matrix (Endüstriyel Standart)

QR kodun daha kompakt ve "ciddi" kuzenidir. Kare veya dikdörtgen olabilir. Köşelerinde QR gibi büyük bloklar yoktur, bunun yerine "L" şeklinde bir bulucu desen kullanır.

- **Kullanım Alanı**: İlaç Endüstrisi (İlaç Takip Sistemi - İTS), elektronik devre kartları, küçük parçaların etiketlenmesi.
- **Özellik**: 2-3 mm boyutuna kadar küçültülebilir. Doğrudan metal üzerine lazerle kazınmaya (DPM - Direct Part Marking) çok uygundur.

### PDF417 (Taşınabilir Veri Dosyası)

Üst üste yığılmış çizgisel barkodlara benzer. Dikdörtgen ve geniş bir yapıdadır.

- **Kullanım Alanı**: Kimlik kartları (ABD ehliyetleri), biniş kartları, lojistik belgeleri.
- **Özellik**: Büyük miktarda metin verisini depolamak için idealdir ancak QR kod kadar hızlı taranmaz.

### Aztec Code

Genellikle ortasında tek bir bulucu kare bulunur.

- **Kullanım Alanı**: Uçak ve tren biletleri.
- **Özellik**: Çevresinde boşluk (quiet zone) bırakılmasına gerek yoktur, bu nedenle kısıtlı alanlarda ve ekranlarda iyi çalışır.

## Teknik Karşılaştırma Tablosu

| Özellik              | QR Code          | Data Matrix      | Code 128         | EAN-13           |
|----------------------|------------------|------------------|------------------|------------------|
| Tür                 | 2D              | 2D              | 1D              | 1D              |
| Veri Kapasitesi     | Çok Yüksek      | Yüksek          | Orta            | Sabit (13 hane) |
| Boyut Verimliliği   | İyi             | Mükemmel (Çok küçük) | İyi         | Orta            |
| Hata Düzeltme       | Var (Güçlü)     | Var (Güçlü)     | Yok (Sadece Check Digit) | Yok (Sadece Check Digit) |
| Okunabilirlik       | 360 Derece      | 360 Derece      | Yatay (Çizgi lazer gerekir) | Yatay      |
| Lisans              | Açık/Ücretsiz   | Açık/Ücretsiz   | Açık/Ücretsiz   | Ücretli (GS1 üyeliği gerekir) |

## Yazılımcılar İçin Kritik İpuçları

### Hangi Kütüphane?

- **Web (Frontend) için**: @yudiel/react-qr-scanner (okuma), qrcode.react (oluşturma).
- **Backend (C#/.NET) için**: ZXing.Net veya SkiaSharp.QrCode.

### Quiet Zone (Sessiz Bölge)

Barkodun etrafındaki boş beyaz alandır. Barkod okuyucunun başlangıç ve bitişi algılaması için bu alana ihtiyacı vardır. Tasarım yaparken barkodu asla bir kutunun veya resmin dibine yapıştırmayın; en az barkod modül genişliğinin 4 katı kadar boşluk bırakın.

### İlaç ve Sağlık Sektörü (Önemli)

Eğer medikal bir proje üzerinde çalışıyorsanız (veterinerlik, eczane vb.), standart GS1 DataMatrix'tir. Bu karekodun içinde sadece ilaç ID'si değil; Seri No, Son Kullanma Tarihi ve Parti No gibi bilgiler özel ayraçlarla (Application Identifiers) saklanır. Standart QR kod okuyucular bunu ham metin olarak okur, ancak anlamlandırmak (parse etmek) için GS1 standartlarını bilmeniz gerekir.

### Renk Kontrastı

Barkodlar kontrast sever. En ideali beyaz zemin üzerine siyah baskıdır. Asla koyu zemin üzerine açık renk barkod (inverted) basmayın; birçok standart okuyucu bunu okuyamaz.

## Sonuç

Bu makalede, barkod ve QR kod türlerinin temel özelliklerini ve kullanım alanlarını inceledik. Projenizde hangi türü seçeceğiniz, veri kapasitesi, okunabilirlik ve sektörel gereksinimlere bağlı olarak değişir. QR Code ve Data Matrix gibi 2D kodlar, modern uygulamalarda daha fazla tercih edilmektedir. Daha detaylı bilgi için ilgili standartları (GS1, ISO) inceleyebilirsiniz.

## EAN ve Code Ayrımı

Ayrımın asıl sebebi **"Kullanım Amacı ve Otorite"** farkıdır. Sadece veri tipi değil, arkasındaki felsefe de farklıdır. İşte EAN ve Code (özellikle Code 128) arasındaki ayrımın ana nedenleri:

### 1. Veri Tipi Farkı (Teknik Sebep)

Evet, haklısınız. Fiziksel olarak kodlama yetenekleri farklıdır:

- **EAN (European Article Number)**: Sadece Rakamları (0-9) destekler. İçine "A" harfini veya "-" işaretini koyamazsınız. Matematiği sadece rakamlar üzerine kuruludur.
- **Code 128**: Alfanümeriktir. Büyük/küçük harf, rakam ve sembolleri (ASCII karakterlerini) destekler.

**Örnek**: A-105 diye bir raf kodunuz varsa, bunu EAN ile yazamazsınız. Mecburen Code 128 (veya Code 39) kullanmalısınız.

### 2. Otorite ve Benzersizlik Farkı (Hukuki Sebep)

Bu en kritik farktır.

- **EAN (Bir Plakadır)**: EAN bir küresel standarttır (GS1). Kafanıza göre "Benim ürünümün barkodu 1234567890123 olsun" diyemezsiniz. EAN kullanmak için GS1 kurumuna para ödeyip firmanıza özel bir ön ek (prefix) alırsınız. Bu sayede o barkod dünyanın her yerinde tekil (unique) olur. Bir market kasası EAN gördüğünde bunun satılabilir bir ticari ürün olduğunu bilir.
- **Code 128 (Bir Not Kağıdıdır)**: Tamamen özgürdür. Kendi deponuzda, kendi yazılımınızda istediğiniz gibi "Stok-01", "Kutu-99" diye barkod basabilirsiniz. Kimse size "Bunu neden kullandın?" demez. Ancak bu kodu alıp Migros'a götürürseniz kasada geçmez, çünkü evrensel bir karşılığı yoktur.

### 3. Yapısal Fark (Boyut Sebebi)

- **EAN Sabittir**: Her zaman 13 hanelidir (EAN-13). Veri az da olsa çok da olsa barkodun genişliği ve yapısı standarttır.
- **Code 128 Esnektir**: 3 karakterlik veri için küçücük, 20 karakterlik veri için upuzun bir barkod oluşur. Veri uzunluğuna göre boyutu değişir.

### Karşılaştırma Tablosu

| Özellik                  | EAN-13 (Ticari)                  | Code 128 (Endüstriyel/Dahili)    |
|--------------------------|----------------------------------|----------------------------------|
| Ne Taşıyabilir?         | Sadece Rakam (0-9)              | Harf, Rakam, Sembol             |
| Kullanım Yeri           | Market, Mağaza (Satış Noktası)  | Kargo, Depo, Personel Kartı, Demirbaş |
| Lisans                  | Paralıdır (GS1 Üyeliği Gerekir) | Ücretsizdir                     |
| Uzunluk                 | Sabit (13 Hane)                 | Değişken (İstediğin kadar uzar) |
| Mantığı                 | "Bu ürünün kimliği nedir?"     | "Bu paketin/nesnenin verisi nedir?" |

### Sizin Projeniz İçin Hangisi?

Veteriner kliniği veya benzeri bir proje geliştirdiğinizi düşünürsek:

- **Marketten aldığınız mamayı satarken**: Ürünün üzerinde zaten üreticinin bastığı bir EAN barkodu vardır. Sizin sisteminiz bu EAN'ı okuyup veritabanından ürünü bulmalıdır.
- **Kendi bastığınız aşı takip kartı veya hasta tasması için**: Code 128 kullanmalısınız. Çünkü buraya hastanın ID'sini (örn: HST-459) gömebilirsiniz. EAN buraya uymaz.

**Özetle**: EAN, "Dünya ile konuşmak" içindir; Code 128, "Kendi sisteminiz veya lojistik ağınızla konuşmak" içindir.