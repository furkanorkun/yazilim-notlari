# Barkod Kullanım Senaryoları: Klinik Yönetim Sistemi İçin Kılavuz

## Giriş

Bu makalede, veteriner kliniği veya benzeri sağlık kurumlarında barkod teknolojisinin nasıl uygulanacağını adım adım inceleyeceğiz. Raf etiketleme, stok yönetimi, ilaç takibi ve demirbaş yönetimi gibi senaryolarda hangi barkod türlerinin kullanılacağını ve teknik detayları ele alacağız.

## AŞAMA 1: Raf ve Lokasyon Etiketleme

Klinik ortamında raflar genellikle dar kenarlı (mobilya tipi veya metal ilaç dolapları) olur. Ayrıca personel elindeki telefonla rafa uzanıp hızlıca okutma yapmak isteyecektir.

### Önerilen Kod Türü: QR Code (Karekod)

Neden klasik çubuk barkod (Code 128) değil de QR Kod?

- **Kamera Odağı (React/Mobile Önceliği)**: Cep telefonu kameraları, yatay ve uzun barkodlara (Code 128) odaklanmakta zorlanır. Özellikle raf kenarına sığdırmak için barkodu küçültürseniz, telefonun netlemesi çok zorlaşır. QR kod karedir, kamera onu merkezine çok daha hızlı alır ve anında okur.
- **Fiziksel Alan**: Raf kenarları genellikle 1.5cm - 2cm yüksekliğindedir.
  - Code 128: İçine "Depo1-Raf3-Bölüm5" yazarsanız, barkod çok uzar (10-15 cm) ve raf kenarına sığmaz.
  - QR Code: Aynı veriyi 1.5cm x 1.5cm bir kareye sığdırabilirsiniz. Rafın kenarına şık durur.
- **Hız**: Personel telefonu yan tutmak veya hizalamak zorunda kalmaz. 360 derece her açıdan okutabilir.

### El Terminali Riski (Önemli Uyarı)

Müşterilerinizden biri "Ben el terminali alacağım" derse, onlara şu teknik şartı koşmalısınız:

*"Alacağınız cihaz kesinlikle '2D Imager' (Karekod Okuyucu) özellikli olmalı. Ucuz '1D Lazer' okuyucular sistemimizdeki raf barkodlarını okumaz."*

Piyasada 2D okuyabilen terminaller artık standartlaştı, yani bu büyük bir sorun değil ama yanlış cihaz almalarını engellemelisiniz.

### Raf Etiketleme Stratejisi (Data Yapısı)

Raf barkodunun içine ne yazacağız? React uygulamanızın veriyi hızlı işlemesi için veriyi kısa ve hiyerarşik tutun. JSON kullanmaya gerek yok (yer kaplar).

**Örnek Veri Formatı**: L-01-R-03-B-02 (Açılımı: Lokasyon 1 (Ana Depo) > Raf 3 > Bölüm 2)

**Prefix (Ön ek) Kullanın**: Barkodun başına ayırt edici bir harf koyun. Örneğin L: veya LOC-.

Neden? Kullanıcı yanlışlıkla raf yerine elindeki ilacı okutursa, yazılımınız şunları ayırt edebilir:

- Okunan veri 869... ile başlıyorsa -> "Bu bir ilaç."
- Okunan veri LOC-... ile başlıyorsa -> "Bu bir raf, ürünleri buraya atayacağım."

### Özet Karar (Raflar İçin)

- **Kod**: QR Code
- **Boyut**: Minimum 1.5cm x 1.5cm
- **İçerik**: Hiyerarşik Düz Metin (Örn: LOC-A-01)
- **Malzeme**: Mümkünse plastik bazlı (yıpranmaz) etiket.

## AŞAMA 2: Stok ve Ürün Yönetimi

Klinik ortamı burayı biraz karmaşıklaştırıyor çünkü elinizde iki farklı dünya var: Market Ürünleri (Mama, tasma vb.) ve Tıbbi Ürünler (İlaç, aşı, enjektör). Bu kategoriyi yönetmek için "Hibrit" bir yaklaşım kullanmalıyız.

### 1. Hazır Barkodlu Ürünler (Mamalar, Aksesuarlar)

Bunlar kutusunda üretici barkoduyla gelir. Bunlar için ekstra etiket basmak zaman ve kağıt israfıdır.

- **Kullanılacak Kod**: EAN-13 (Üzerinde gelen barkod).
- **Strateji**: Sisteminizde ürün kartını oluştururken, elinizdeki telefonla ürünün arkasındaki EAN barkodunu okutup "Bu barkod X mamasına aittir" diye eşleştirme yapacaksınız.
- **Mobil Uygulama Notu**: @yudiel/react-qr-scanner kütüphanesine `formats: ['ean_13', 'qr_code']` parametresini vererek hem rafı (QR) hem ürünü (EAN) aynı anda desteklemesini sağlayacaksınız.

### 2. İlaçlar ve Aşılar (Kritik Nokta: İTS ve Data Matrix)

Türkiye'de ve dünyada ilaçların üzerinde standart barkod (çizgili) değil, Karekod (Data Matrix) bulunur. Bu bir QR kod değildir, Data Matrix'tir.

- **Kullanılacak Kod**: GS1 Data Matrix (Üzerinde gelen).
- **Neden Önemli?** İlaç kutusundaki o küçük karenin içinde sadece ürünün ID'si yazmaz. Şunlar yazar:
  - GTIN (Ürün Kodu): Hangi ilaç olduğu.
  - SKT (Son Kul. Tar.): İlacın ne zaman bozulacağı.
  - Parti No (Lot): Hangi üretim serisi olduğu.
  - Seri No: O kutunun dünyadaki tekil kimliği.
- **Yazılım Stratejisi**: Senin React uygulaman bu karekodu okuduğunda uzun bir metin dizisi (string) alacak (Örn: 010869...17251231...). Yazılımın arka planda bunu parçalayıp (parse edip) SKT'yi otomatik olarak sisteme kaydetmeli. Bu, klinik yönetiminde "Süresi dolan ilaçları uyar" özelliği için hayati önem taşır.

### 3. Barkodsuz veya Bölünen Ürünler (Tane İlaçlar, Kitler)

Kliniklerde en büyük sorun budur. 100'lük kutudan 1 adet hap satarsınız veya barkodsuz bir cerrahi ip kullanırsınız. Bunlara etiket basmanız gerekir.

- **Önerilen Kod Türü**: QR Code (veya Micro QR).
- **Neden Code 128 değil?** Bir ampul veya enjektör çok ince ve küçüktür. Çizgili barkodu (Code 128) silindirik bir yüzeye (şişeye) yapıştırırsanız, barkod bükülür. Kamera bükülen çizgileri okuyamaz. QR Kod küçüktür (1cm x 1cm) ve bükülme sorunu daha azdır.
- **Veri Formatı**: Kendi ürettiğiniz bu etiketlere yine bir ön ek (Prefix) koyun. Örn: INT-102030 (INT: Internal/Dahili).

### Özet Tablo (Stok Senaryosu)

| Ürün Tipi          | Barkod Türü   | Eylem                     | Mobil Uygulama Ayarı |
|--------------------|---------------|---------------------------|----------------------|
| Ticari (Mama/Tasma) | EAN-13       | Mevcut olanı okut         | `formats: ['ean_13']` |
| İlaç/Aşı           | Data Matrix  | Mevcut olanı okut (Parse et) | `formats: ['data_matrix']` |
| Dahili/Bölünmüş    | QR Code      | Etiket Yazıcıdan Bas      | `formats: ['qr_code']` |

## EK PARANTEZ: İlaç Karekodunu (Data Matrix) Nasıl Ayrıştıracaksınız?

Data Matrix ayrıştırma (parsing) işlemi, barkod dünyasının en teknik ve hata yapılan kısmıdır. React tarafında bunu nasıl yöneteceğinizi aşağıda özetledim ve ardından son aşama olan Demirbaş'a geçiyoruz.

İlacı okuttuğunuzda @yudiel/react-qr-scanner size şöyle karmaşık, tek satırlık bir metin (string) döndürecektir:

```
01086995460205321725123110A345B2110000555
```

Bunu veritabanına doğrudan kaydetmemelisiniz. Arka planda şu mantıkla parçalamanız gerekir (GS1 Standartları):

- **01 (GTIN - Ürün Kimliği)**: Baştaki 01 tanımlayıcısını görünce, sonraki 14 haneyi alırsınız. Bu, ilacın barkod numarasıdır (EAN'ın 14 haneli halidir).
- **17 (SKT)**: Sonraki 17 tanımlayıcısını görünce, takip eden 6 haneyi alırsınız. Formatı YYAAGG şeklindedir. Örnekteki 251231 -> 31 Aralık 2025.
- **10 (Parti No) ve 21 (Seri No)**: Bunlar değişken uzunluktadır. GS1 standardında bu alanların bittiğini anlamak için arada görünmeyen özel bir karakter (ASCII 29 - Group Separator) kullanılır. Ancak web tarayıcıları bazen bu karakteri "yutar" veya boşluğa çevirir. Yazarken buna dikkat etmeniz gerekecek (Regex kullanmak en iyisidir).

**Teknik Not**: React uygulamanızda scanner ayarına mutlaka `formats: ['data_matrix', ...]` eklemeyi unutmayın.

## AŞAMA 3: Demirbaş Yönetimi (Bilgisayarlar, Cihazlar, Mobilyalar)

Klinikteki ultrason cihazı, bilgisayarlar, sandalyeler vb. satılmaz ama zimmetlenir ve sayılır. Bunlar yıllarca durur, tozlanır, silinir.

### Önerilen Kod Türü: QR Code

Burada da Code 128 yerine QR Code kullanmanızı şiddetle öneririm.

- **Yıpranma ve Dayanıklılık**: Demirbaş etiketleri genellikle cihazların arkasına veya mobilyaların altına yapıştırılır. Zamanla çizilirler. QR kodun hata düzeltme (Error Correction) özelliği sayesinde, etiketin %20-30'u silinse veya kirlense bile okuyabilirsiniz. Code 128'in bir çizgisi silinirse o barkod çöp olur.
- **Boyut ve Estetik**: Pahalı bir tıbbi cihazın üzerine kocaman bir barkod yapıştırmak istemezsiniz. 1cm x 1cm'lik gümüş renkli, şık bir QR etiket profesyonel durur.
- **Tek Tip Okuma Deneyimi**: Personel rafları QR ile okuyor, ilaç dışı ürünleri QR ile okuyor. Demirbaş için de QR kullanırsanız, kamera modunu değiştirmeye gerek kalmaz.

### Veri Yapısı (Data Structure)

Demirbaşlar için "Dahili Barkod" mantığını kullanın ve mutlaka benzersiz bir ön ek (Prefix) verin.

**Örnek**: AST-2024-001 (Asset - Yıl - Sıra No)

**Prefix Mantığı**: Yazılımınız AST ile başlayan bir kod okuduğunda; "Bu bir ürün değil, bu bir demirbaş. Satış ekranında bunu gösterme, zimmet ekranını aç" diyebilir.

## ÖZET: Sistemin Büyük Resmi (Grand Design)

Klinik yönetim sistemi için "Master Plan" şu şekilde oluştu:

| Kullanım Yeri       | Seçilen Teknoloji | Veri Kaynağı     | Okuyucu Format Ayarı | Örnek Veri          |
|---------------------|-------------------|------------------|-----------------------|---------------------|
| Raflar             | QR Code          | Biz basacağız   | `qr_code`            | LOC-A-05           |
| Ticari Ürünler (Mama vb.) | EAN-13          | Üzerinde hazır  | `ean_13`             | 8691234567890      |
| İlaç & Aşı         | Data Matrix      | Üzerinde hazır  | `data_matrix`        | 010869...1725... (Parse edilecek) |
| Bölünen Ürünler    | QR Code          | Biz basacağız   | `qr_code`            | INT-999             |
| Demirbaşlar        | QR Code          | Biz basacağız   | `qr_code`            | AST-005             |

### Final React Uygulaması Mimarisi

Artık yazacağın React bileşeninde (Scanner Component) tek bir strateji izleyeceksin:

1. **Scanner'ı Başlat**: formats dizisine `['qr_code', 'data_matrix', 'ean_13']` ver.
2. **Veriyi Yakala**: onScan olayında gelen veriyi al.
3. **Yönlendir (Router Logic)**:
   - if (format === 'ean_13') -> Ürün veritabanında ara.
   - if (format === 'data_matrix') -> GS1 Parser fonksiyonuna gönder -> SKT'yi ayıkla -> Ürün veritabanında ara.
   - if (format === 'qr_code') -> Verinin başına bak:
     - LOC- ile başlıyorsa -> Raf detayını getir.
     - AST- ile başlıyorsa -> Demirbaş detayını getir.
     - INT- ile başlıyorsa -> Dahili ürün detayını getir.

## Sonuç

Bu makalede, klinik yönetim sisteminde barkod teknolojisinin uygulanması için kapsamlı bir kılavuz sunduk. QR Code'un mobil uyumluluğu, Data Matrix'in ilaç takibi için önemi ve EAN-13'ün ticari ürünler için kullanımı gibi detayları ele aldık. Bu yaklaşım, sisteminizin verimliliğini artırırken hataları minimize edecektir. Daha detaylı bilgi için GS1 standartlarını inceleyebilirsiniz.

## Ek: Tam Kapsamlı React Uygulaması Örneği

Konuştuğumuz tüm senaryoları (Raf, Demirbaş, İlaç/Karekod, Market Ürünü) tek bir merkezde yöneten, Clean Architecture prensiplerine uygun olarak mantığı ayrıştırılmış tam kapsamlı bir React yapısı hazırladım.

Bu kod şunları yapar:

- **Ayrıştırma (Routing)**: Okunan kodun formatına ve içeriğine bakarak ne olduğunu anlar.
- **GS1 Parsing**: İlaç karekodlarını parçalar (SKT ve Parti No çıkarır).
- **Prefix Kontrolü**: Raf mı, demirbaş mı ayırt eder.

### 1. Yardımcı Fonksiyon: GS1 DataMatrix Parser

Önce ilaç barkodlarını çözecek yardımcı fonksiyonu yazalım. Bunu ayrı bir dosya (örn: utils/barcodeParser.js) olarak düşünebilirsin.

```javascript
// utils/barcodeParser.js

/**
 * İlaç Karekodunu (GS1 DataMatrix) parçalar.
 * Desteklenen AI (Application Identifiers):
 * 01: GTIN (Ürün Kodu) - 14 hane
 * 17: SKT (Son Kul. Tar.) - 6 hane (YYAAGG)
 * 10: Parti No (Batch) - Değişken uzunluk
 */
export const parseMedicalBarcode = (rawValue) => {
  let data = {
    gtin: null,
    expiryDate: null,
    batchNo: null,
    serialNo: null,
    raw: rawValue
  };

  // 1. GTIN (01) yakala: Genellikle baştadır ve sabittir.
  // Not: Bazı barkodlarda parantez (01) olabilir, temizlemek gerekebilir.
  const cleanRaw = rawValue.replace(/[()]/g, ''); 

  // Basit ve güvenli parsing (Regex yerine substring mantığı daha stabil olabilir)
  // GS1 standardında AI'lar bellidir.
  
  let currentIndex = 0;

  // Başlangıçta 01 var mı?
  if (cleanRaw.startsWith('01')) {
    data.gtin = cleanRaw.substring(2, 16); // 2'den başla, 14 karakter al
    currentIndex = 16;
  }

  // Geri kalan string içinde diğerlerini ara
  const remaining = cleanRaw.substring(currentIndex);
  
  // SKT (17) Bul (Genellikle GTIN'den sonra gelir)
  // Not: Gerçek hayatta bu sıra değişebilir, regex daha güvenli olabilir ama
  // Türkiye İlaç sisteminde genelde sıralı gelir: 01...21...17...10...
  
  // Daha robust bir Regex yaklaşımı:
  const gtinMatch = cleanRaw.match(/01(\d{14})/);
  const expiryMatch = cleanRaw.match(/17(\d{6})/);
  const batchMatch = cleanRaw.match(/10([A-Z0-9]+)/); 
  // Not: Batch match sonundaki Group Separator (<GS>) karakterine dikkat etmek gerekir.
  // Web scannerlar bazen GS karakterini yutar, bu yüzden sonuna kadar alırız.

  if (gtinMatch) data.gtin = gtinMatch[1];
  if (expiryMatch) {
    const rawDate = expiryMatch[1];
    // Tarihi okunabilir formata çevir (251231 -> 31/12/2025)
    data.expiryDate = `20${rawDate.substring(0,2)}-${rawDate.substring(2,4)}-${rawDate.substring(4,6)}`;
  }
  if (batchMatch) data.batchNo = batchMatch[1];

  return data;
};
```

### 2. Ana Bileşen: SmartScanner

Şimdi bu mantığı React içinde kullanalım.

```javascript
// components/SmartScanner.jsx
import React, { useState } from 'react';
import { Scanner } from '@yudiel/react-qr-scanner';
import { parseMedicalBarcode } from '../utils/barcodeParser'; // Yukarıdaki dosya

const SmartScanner = () => {
  const [scanResult, setScanResult] = useState(null);
  const [error, setError] = useState('');

  const handleScan = (detectedCodes) => {
    if (!detectedCodes || detectedCodes.length === 0) return;

    const code = detectedCodes[0];
    const { rawValue, format } = code;

    // --- ANA YÖNLENDİRME MANTIĞI (ROUTER) ---
    let processResult = {
      type: 'UNKNOWN',
      displayTitle: 'Bilinmeyen Kod',
      data: rawValue
    };

    // SENARYO 1: İLAÇ / AŞI (Data Matrix)
    if (format === 'data_matrix') {
      const medicalData = parseMedicalBarcode(rawValue);
      processResult = {
        type: 'MEDICAL',
        displayTitle: '💊 İlaç / Aşı Tespit Edildi',
        data: medicalData
      };
    }
    
    // SENARYO 2: MARKET ÜRÜNÜ (EAN-13)
    else if (format === 'ean_13') {
      processResult = {
        type: 'COMMERCIAL',
        displayTitle: '🛒 Market Ürünü',
        data: { barcode: rawValue }
      };
    }

    // SENARYO 3: QR KODLAR (Raf, Demirbaş, Dahili)
    else if (format === 'qr_code') {
      
      // Prefix Kontrolü
      if (rawValue.startsWith('LOC-')) {
        processResult = {
          type: 'LOCATION',
          displayTitle: '📍 Raf / Lokasyon',
          data: { locationId: rawValue }
        };
      } 
      else if (rawValue.startsWith('AST-')) {
        processResult = {
          type: 'ASSET',
          displayTitle: '💻 Demirbaş',
          data: { assetId: rawValue }
        };
      }
      else if (rawValue.startsWith('INT-')) {
        processResult = {
          type: 'INTERNAL_PRODUCT',
          displayTitle: '📦 Dahili / Bölünmüş Ürün',
          data: { productId: rawValue }
        };
      }
      else {
        // Prefixsiz, sıradan bir QR kod (Örn: Web sitesi)
        processResult = {
          type: 'GENERAL_QR',
          displayTitle: '🔗 Genel QR Kod',
          data: { content: rawValue }
        };
      }
    }

    // Sonucu State'e at (Burada API isteği de atılabilir)
    setScanResult(processResult);
  };

  return (
    <div style={{ maxWidth: '500px', margin: '0 auto' }}>
      <h2>Klinik Barkod Tarayıcı</h2>
      
      <div style={{ border: '2px solid #ddd', borderRadius: '10px', overflow: 'hidden' }}>
        <Scanner
          onScan={handleScan}
          onError={(err) => setError(err?.message)}
          // KRİTİK AYAR: Tüm formatları açıyoruz
          formats={['qr_code', 'data_matrix', 'ean_13']}
          // Arka arkaya taramayı engellemek için gecikme (ms)
          scanDelay={2000} 
        />
      </div>

      {error && <p style={{ color: 'red' }}>Hata: {error}</p>}

      {/* SONUÇ EKRANI */}
      {scanResult && (
        <div style={{ 
          marginTop: '20px', 
          padding: '15px', 
          backgroundColor: '#f0f9ff', 
          border: '1px solid #bae6fd',
          borderRadius: '8px' 
        }}>
          <h3>{scanResult.displayTitle}</h3>
          
          {/* İLAÇ DETAYI GÖSTERİMİ */}
          {scanResult.type === 'MEDICAL' && (
            <ul>
              <li><strong>GTIN:</strong> {scanResult.data.gtin}</li>
              <li><strong>SKT:</strong> {scanResult.data.expiryDate}</li>
              <li><strong>Parti:</strong> {scanResult.data.batchNo}</li>
            </ul>
          )}

          {/* DİĞER TİPLER */}
          {scanResult.type !== 'MEDICAL' && (
            <pre style={{ background: '#fff', padding: '10px' }}>
              {JSON.stringify(scanResult.data, null, 2)}
            </pre>
          )}

          <button 
            onClick={() => setScanResult(null)}
            style={{ marginTop: '10px', padding: '5px 10px', cursor: 'pointer' }}
          >
            Yeni Tarama Yap
          </button>
        </div>
      )}
    </div>
  );
};

export default SmartScanner;
```

### Kodun Çalışma Mantığı

- **Scanner Config**: `formats={['qr_code', 'data_matrix', 'ean_13']}` satırı sayesinde kamera aynı anda her şeye bakıyor.
- **Logic Tree (Karar Ağacı)**:
  - Önce format'a bakıyor.
  - Eğer qr_code ise içeriğin başındaki metne (`startsWith`) bakıyor.
  - Eğer data_matrix ise `parseMedicalBarcode` fonksiyonuna gönderip anlamlı veri çıkartıyor.
- **JSON Output**: Sonuçta eline tertemiz bir JSON objesi geçiyor. Bunu doğrudan backend'ine (POST /api/action) gönderebilirsin.