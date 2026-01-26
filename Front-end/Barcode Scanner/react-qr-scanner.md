# React QR Scanner Kütüphanesi: @yudiel/react-qr-scanner

## Giriş

@yudiel/react-qr-scanner, React ekosisteminde QR kod ve barkod okuma işlemleri için en popüler, güncel ve bakımı yapılan kütüphanelerden biridir. Bu makalede, kütüphanenin özelliklerini, kurulumunu ve kullanımını detaylı bir şekilde inceleyeceğiz.

## Neden @yudiel/react-qr-scanner Kullanmalısınız?

Bu kütüphane, modern web uygulamalarında barkod tarama ihtiyaçlarını karşılamak için ideal bir çözümdür. İşte başlıca avantajları:

- **Modern ve Güncel**: React Hooks yapısına tam uyumludur ve fonksiyonel bileşenlerle sorunsuz çalışır.
- **Çapraz Tarayıcı Desteği**: Native Barcode API'nin aksine, Safari (iOS) dahil hemen hemen tüm modern tarayıcılarda sorunsuz çalışır.
- **Hafif ve Esnek**: Özelleştirmesi kolaydır ve responsive bir yapıya sahiptir.
- **Çoklu Format Desteği**: Sadece QR kod değil; Code 128, EAN, UPC gibi çeşitli barkod türlerini okuyabilir.

## Kurulum

Kütüphaneyi projenize dahil etmek için aşağıdaki komutlardan birini kullanabilirsiniz:

```bash
pnpm add @yudiel/react-qr-scanner
# veya
npm install @yudiel/react-qr-scanner
# veya
yarn add @yudiel/react-qr-scanner
```

## Temel Kullanım

Kütüphanenin en güncel sürümünde (v2.x) temel bileşen `<Scanner />` olarak adlandırılır. Aşağıda basit bir kullanım örneği verilmiştir:

```javascript
import { Scanner } from '@yudiel/react-qr-scanner';

const MyQrComponent = () => {
    // Tarama başarılı olduğunda çalışacak fonksiyon
    const handleScan = (result) => {
        if (result) {
            // result bir dizidir, genellikle ilk eleman kullanılır
            console.log('Okunan Değer:', result[0].rawValue);
            console.log('Format:', result[0].format);
        }
    };

    return (
        <div style={{ width: '100%', maxWidth: '400px' }}>
            <Scanner
                onScan={handleScan}
                onError={(error) => console.log(error?.message)}
            />
        </div>
    );
};

export default MyQrComponent;
```

## Önemli Props ve Ayarlar

Uygulamanızı özelleştirmek için kullanabileceğiniz kritik parametreler şunlardır:

| Prop          | Açıklama                                                                 |
|---------------|--------------------------------------------------------------------------|
| `onScan`     | Barkod algılandığında tetiklenir. Dizi döner. En önemli prop budur.     |
| `onError`    | Kamera izni verilmediğinde veya hata oluştuğunda çalışır.               |
| `formats`    | Hangi barkod türlerinin taranacağını belirtir. Örn: `['qr_code', 'ean_13']` |
| `constraints` | Hangi kameranın kullanılacağını seçer. Örn: `{ facingMode: 'environment' }` |
| `scanDelay`  | İki tarama arasındaki bekleme süresi (ms).                              |
| `components` | UI elemanlarını açıp kapatmanızı sağlar. Örn: `{ audio: false, tracker: true }` |
| `allowMultiple` | `true` yapılırsa aynı barkodu peş peşe okumaya izin verir.             |

## Kritik Teknik Detaylar

Bu kütüphaneyi kullanırken karşılaşabileceğiniz yaygın sorunlar ve çözümleri:

- **HTTPS Zorunluluğu**: Tarayıcılar kamera erişimine sadece HTTPS veya localhost üzerinde izin verir.
- **Kapsayıcı Boyutu**: Scanner bileşeni, kapsayıcı div'in boyutunu alır. Yüksekliğin belirlenmesi önemlidir.
- **React Strict Mode**: Geliştirme ortamında kamera sorunları yaşayabilirsiniz; production'da sorun olmaz.
- **Performans İpucu**: Sadece QR kod için `formats={['qr_code']}` kullanın.

## Native API vs. @yudiel/react-qr-scanner

| Özellik                  | Native Barcode Detection API          | @yudiel/react-qr-scanner              |
|--------------------------|---------------------------------------|---------------------------------------|
| Tarayıcı Desteği        | Sınırlı (Safari yok)                 | Mükemmel (Tüm modern tarayıcılar)    |
| Performans               | Çok Yüksek                           | İyi                                  |
| Bundle Boyutu            | 0 KB                                 | Projenize boyut ekler                |
| Kullanım Kolaylığı      | Orta                                 | Çok Kolay                            |

## Özet ve Tavsiye

Eğer projeniz halka açık bir web sitesi veya PWA ise ve kullanıcıların cihaz çeşitliliğini göz önünde bulunduruyorsanız, kesinlikle @yudiel/react-qr-scanner kullanın. Native API henüz iOS tarafında yeterli desteği sağlamadığı için bu kütüphane en güvenli seçenektir.

Bu makalede, kütüphanenin temel özelliklerini ve kullanımını ele aldık. Daha detaylı bilgi için resmi dokümantasyonu inceleyebilirsiniz.