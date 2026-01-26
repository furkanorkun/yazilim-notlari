Barcode Detection API (Barkod Algılama API'si), modern web tarayıcılarında harici bir JavaScript kütüphanesine (library) ihtiyaç duymadan, doğrudan tarayıcı yeteneklerini kullanarak barkod ve QR kodlarını okumanıza olanak tanıyan deneysel ama güçlü bir web standardıdır. Harici kütüphaneler kullanmandan işlemi donanım hızlandırmalı (hardware-accelerated) olarak yaparak performansı artırır ve uygulamanızın boyutunu küçültür.

## Temel Özellikleri

| Özellik | Açıklama |
|---------|----------|
| **Performans** | İşletim sisteminin veya tarayıcının yerel (native) görüntü işleme yeteneklerini kullanır. Bu, JavaScript tabanlı çözümlerden çok daha hızlıdır ve CPU'yu daha az yorar. |
| **Format Desteği** | Sadece QR kodlarını değil, market barkodları (EAN), kargo barkodları (Code 128) gibi birçok formatı destekler. |
| **Kolay Entegrasyon** | Karmaşık görüntü işleme algoritmaları yazmanıza gerek kalmaz; sadece bir görüntü kaynağı (resim, video veya canvas) verirsiniz. |

## Desteklenen Formatlar

API, getSupportedFormats() metodu ile o anki cihazın hangi formatları desteklediğini size söyleyebilir.

| Format Tipi | Desteklenen Formatlar |
|-------------|----------------------|
| **2D Kodlar** | qr_code, aztec, data_matrix, pdf417 |
| **1D (Çizgisel) Kodlar** | ean_13, ean_8, upc_a, upc_e, code_128, code_39, code_93, itf, codabar |

## Nasıl Kullanılır? (Örnek Kod)

API'nin kullanımı oldukça basittir. BarcodeDetector sınıfını başlatırsınız ve detect metoduna bir görüntü kaynağı verirsiniz.

```JavaScript
// 1. Özellik kontrolü yapın
if ('BarcodeDetector' in window) {
  
  // 2. Dedektörü başlatın (Opsiyonel: Okunacak formatları belirtin)
  const barcodeDetector = new BarcodeDetector({
  formats: ['qr_code', 'ean_13', 'code_128']
  });

  // 3. Bir kaynağı tarayın (img, video veya canvas olabilir)
  const imageElement = document.getElementById('myImage');

  barcodeDetector.detect(imageElement)
  .then(barcodes => {
    barcodes.forEach(barcode => {
    console.log('Okunan Değer:', barcode.rawValue);
    console.log('Format:', barcode.format);
    console.log('Konum (Bounding Box):', barcode.boundingBox);
    });
  })
  .catch(err => {
    console.error('Barkod algılama hatası:', err);
  });

} else {
  console.log('Tarayıcınız Barcode Detection API desteklemiyor.');
  // Burada "polyfill" veya harici kütüphane devreye girmeli
}
```

## Tarayıcı Desteği ve Uyumluluk (Kritik Nokta)

Bu API henüz tam standartlaşmamış olabilir ve tarayıcı desteği değişkendir.

| Tarayıcı | Destek Durumu |
|----------|---------------|
| **Google Chrome / Edge / Opera** | Masaüstü ve Android üzerinde genellikle tam destek verir (Chrome 83+). |
| **Safari (iOS/macOS)** | Şu an için desteklenmiyor. Apple, kendi Vision framework'ü üzerinden benzer yeteneklere sahip olsa da, Web API olarak bunu henüz Safari'ye açmadı. |
| **Firefox** | Varsayılan olarak kapalıdır veya desteklenmez. |

**Önemli Not:** Bu API'yi kullanacaksanız, mutlaka Progressive Enhancement (Aşamalı Geliştirme) yapmalısınız. Yani API varsa bunu kullanmalı, yoksa kütüphanelere düşmelisiniz (fallback).

## Avantajlar ve Dezavantajlar

| Avantajlar (Pros) | Dezavantajlar (Cons) |
|-------------------|----------------------|
| Hız: Native çalıştığı için çok hızlıdır. | Uyumluluk: Safari ve Firefox desteği eksiktir. |
| Boyut: Harici kütüphane yüklemediğiniz için JS bundle boyutunuz küçülür. | Tutarlılık: Desteklenen barkod formatları cihazdan cihaza (Android sürümü, donanım vb.) değişebilir. |
| Kullanım Kolaylığı: Basit ve asenkron (Promise tabanlı) bir API yapısı vardır. | |
| Konum Tespiti: Barkodun görüntünün neresinde olduğunu (koordinatlarını) verir. | |

## Kullanım Alanları
* Hızlı Ödeme Sistemleri: QR kod ile ödeme alma.
* Stok Takip: Web tabanlı depo yönetim sistemlerinde ürün barkodu okutma.
* Bilet Kontrol: Etkinlik girişlerinde bilet üzerindeki kodları tarama.
* Fatura Ödeme: Fatura üzerindeki barkodları taratarak veri girişi yapma.

## Özet: Ne Yapmalısınız?
Eğer bir web projesinde barkod okuma özelliği geliştirecekseniz stratejiniz şu olmalı:
1. Öncelikle window.BarcodeDetector var mı diye kontrol edin.
2. Varsa: Native API'yi kullanın (Kullanıcıya en iyi performansı sunarsınız).
3. Yoksa (örneğin iOS Safari): html5-qrcode gibi popüler ve hafif bir kütüphaneyi devreye sokun (Polyfill mantığı).