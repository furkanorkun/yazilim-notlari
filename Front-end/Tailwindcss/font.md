# Font Yönetimi Rehberi

Bir React projesinde font yönetimi iki ana yoldan yapılır: Ya npm üzerinden yönetilen paketler (@fontsource) ya da manuel dosya yönetimi. Her iki yöntem de Google Fonts CDN'ine göre daha performanslıdır çünkü fontlar uygulamanızla birlikte sunulur (Self-hosted), DNS sorgusu ve harici bağlantı gecikmesi yaşanmaz.

## BÖLÜM 1: Endüstri Standardı - @fontsource Kullanımı

Açık kaynak fontlar (Google Fonts kütüphanesindeki fontlar: Inter, Roboto, Poppins vb.) kullanıyorsanız, dosyaları tek tek indirmek yerine @fontsource kullanmak en temiz yöntemdir.

### 1.1. Variable Font vs. Statik Font

@fontsource artık Variable Font (Değişken Font) desteğine odaklanmaktadır.

- **Statik:** Her ağırlık (400, 500, 700) ayrı bir dosyadır. 3 ağırlık kullanırsanız 3 dosya indirilir.
- **Variable:** Tek bir dosya tüm ağırlıkları ve bazen eğiklikleri (slant) içerir. Tarayıcı tek istek atar. Önerilen budur.

### 1.2. Kurulum (Örnek: Inter)

**Senaryo A: Modern Yöntem (Variable Font)** Tek bir paket ile tüm ağırlıklara sahip olursunuz.

```bash
npm install @fontsource-variable/inter
```

**Senaryo B: Klasik Yöntem (Spesifik Ağırlıklar)** Eğer fontun variable desteği yoksa klasik paket kullanılır.

```bash
npm install @fontsource/inter
```

### 1.3. React Entegrasyonu

Fontu projenin en tepesindeki dosyada (genellikle main.tsx veya App.tsx) import etmeniz gerekir.

```javascript
// main.tsx

// A) Variable kullanıyorsanız (Sadece bunu import etmek yeterli)
import '@fontsource-variable/inter';

// B) Statik kullanıyorsanız (Sadece kullandığınız ağırlıkları seçin)
// import '@fontsource/inter/400.css';
// import '@fontsource/inter/500.css';
// import '@fontsource/inter/700.css';
```

## BÖLÜM 2: Tam Kontrol - Manuel (Self-Hosted) Kurulum

Özel bir lisanslı fontunuz varsa (Örn: Axiforma, Helvetica vb.) veya @fontsource üzerinde olmayan bir font kullanıyorsanız bu yöntemi kullanmalısınız.

### 2.1. Altın Kural: WOFF2 Formatı

Elinizdeki .ttf veya .otf dosyalarını kullanmayın. Web için optimize edilmiş sıkıştırma formatı WOFF2'dir. Dosyalarınızı Transfonter gibi araçlarla WOFF2'ye çevirin.

### 2.2. Profesyonel Klasör Yapısı

Dosyaları public klasörüne değil, src/assets altına koyun. Vite, assets altındaki dosyalara build sırasında benzersiz hash'ler ekler (cache-busting), böylece fontu güncellediğinizde kullanıcılar eski versiyonu görmez.

```
src/assets/fonts/Axiforma/
  ├── Axiforma-Book.woff2
  ├── Axiforma-Regular.woff2
  ├── Axiforma-Bold.woff2
  └── Axiforma-Heavy.woff2
```

### 2.3. Modüler CSS Yönetimi (fonts.css)

Font tanımlarını ana CSS dosyasını kirletmemek için ayrı bir dosyada yapın.

**Kritik Teknik Detay:** Tüm varyasyonlara aynı font-family ismini verin. Tarayıcı font-weight değerine göre doğru dosyayı kendisi seçecektir.

```css
/* src/assets/fonts/fonts.css */

/* Book (350) - Ara Değer */
@font-face {
  font-family: 'Axiforma';
  src: url('./Axiforma/Axiforma-Book.woff2') format('woff2');
  font-weight: 350;
  font-style: normal;
  font-display: swap; /* Font yüklenene kadar sistem fontunu göster */
}

/* Regular (400) */
@font-face {
  font-family: 'Axiforma';
  src: url('./Axiforma/Axiforma-Regular.woff2') format('woff2');
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}

/* Bold (700) */
@font-face {
  font-family: 'Axiforma';
  src: url('./Axiforma/Axiforma-Bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

/* Heavy (850) - Ara Değer */
@font-face {
  font-family: 'Axiforma';
  src: url('./Axiforma/Axiforma-Heavy.woff2') format('woff2');
  font-weight: 850;
  font-style: normal;
  font-display: swap;
}
```

## BÖLÜM 3: Tailwind CSS v4 Entegrasyonu

Tailwind v4 ile birlikte JavaScript tabanlı yapılandırma (tailwind.config.js) yerini CSS-First yapılandırmaya bıraktı. Artık fontları CSS değişkenleri (Variables) üzerinden yönetiyoruz.

### 3.1. Font Ailesini Tanımlama ve Ezme

Tailwind'in varsayılan sans ailesini (Inter, Roboto vs. içeren o uzun listeyi) kendi fontumuzla güncellemek istiyoruz.

src/index.css dosyanızı açın:

```css
@import "tailwindcss";
@import "./assets/fonts/fonts.css"; /* Manuel fontlarımızı dahil ettik */

@theme {
  /* 1. Tailwind'in varsayılan "font-sans" utility'sini güncelliyoruz.
     En başa kendi fontumuzu (Axiforma veya Inter) yazıyoruz.
     Devamına "System Stack" ekliyoruz ki font yüklenmezse site bozulmasın.
  */
  --font-sans: "Axiforma", "Inter", ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";

  /*
     2. Eğer fontsource kullanıyorsanız 'Axiforma' yerine
     'Inter Variable' veya 'Inter' yazmanız yeterlidir.
  */
}
```

### 3.2. Özel Ağırlıkları (Custom Weights) Yönetmek

Standart Tailwind sınıfları font-thin (100) ile font-black (900) arasındadır. Ancak Axiforma örneğindeki gibi Book (350) veya Heavy (850) gibi ara değerleriniz varsa bunları sisteme tanıtmanız gerekir.

Yine src/index.css içerisindeki @theme bloğuna ekleme yapıyoruz:

```css
@theme {
  /* ...font-sans tanımı... */

  /* Yeni Utility Classlar Üretiyoruz */
  --font-weight-book: 350;
  --font-weight-heavy: 850;
}
```

**Sonuç:** Tailwind bu tanımları gördüğünde otomatik olarak şu sınıfları oluşturur:

- `.font-book` → `font-weight: 350;`
- `.font-heavy` → `font-weight: 850;`

### 3.3. Kullanım Örneği

```javascript
export default function HeroSection() {
  return (
    <div className="font-sans"> {/* Otomatik Axiforma oldu */}
      
      {/* 850 Ağırlığında Başlık */}
      <h1 className="font-heavy text-4xl">
        Veteriner Kliniği Yönetimi
      </h1>
      
      {/* 350 Ağırlığında Alt Metin */}
      <p className="font-book text-gray-600 mt-2">
        Modern ve güvenilir çözüm ortağınız.
      </p>

      {/* Standart 700 Ağırlığında Buton */}
      <button className="font-bold bg-blue-500 text-white px-4 py-2">
        Giriş Yap
      </button>
    </div>
  )
}
```

## BÖLÜM 4: Performans İpuçları (Best Practices)

Fontları projeye eklemek yetmez, hızlı yüklenmelerini sağlamak gerekir.

### 4.1. Preloading (Ön Yükleme)

Tarayıcı, CSS'i okuyup "Bu sayfada Bold fonta ihtiyaç var" diyene kadar fontu indirmez. Bu gecikmeyi önlemek için, sadece ekranın ilk açılışında görünen (Above the fold) ana fontları index.html içinde preload edin.

```html
<link rel="preload" href="/src/assets/fonts/Axiforma/Axiforma-Regular.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/src/assets/fonts/Axiforma/Axiforma-Bold.woff2" as="font" type="font/woff2" crossorigin>
```

### 4.2. font-display: swap

Bu özellik Core Web Vitals (LCP ve CLS) skorları için kritiktir.

- **Swap:** Font yüklenene kadar sistem fontunu gösterir, yüklenince değiştirir. Yazı her zaman okunabilir kalır.
- **Block:** Font yüklenene kadar yazıyı gizler. (Önerilmez, kullanıcı boş sayfa görür).

### 4.3. Fallback Font Uyumu

Eğer özel fontunuz (Axiforma) ile sistem fontu (Arial/Sans-serif) arasında ciddi boyut farkı varsa, font yüklendiğinde sayfa zıplama yapar (Layout Shift). Bunu engellemek için size-adjust CSS özelliği kullanılabilir ancak bu ileri seviye bir konudur. Genellikle Tailwind config içindeki fallback listesini (ui-sans-serif, system-ui...) doğru sıralamak yeterlidir.

## Özet Karşılaştırma

| Özellik | @fontsource (Inter vb.) | Manuel Self-Hosted (Axiforma vb.) |
|---------|------------------------|----------------------------------|
| Kurulum | Çok Kolay (npm i) | Orta (Dosya yönetimi gerekir) |
| Performans | Mükemmel (Otomatik optimize) | Mükemmel (Doğru yapılandırılırsa) |
| Esneklik | Sadece kütüphanedeki fontlar | Her türlü font |
| Tailwind v4 | CSS değişkeni ile isim atanır | CSS değişkeni + Özel ağırlıklar |
| Dosya Boyutu | Variable font ile çok düşük | WOFF2 dönüşümü şart |

> **İpucu:** Font yönetimi performans ve kullanıcı deneyimi açısından kritiktir. Her zaman WOFF2 formatını kullanın ve font-display: swap özelliğini unutmayın.
Tailwind v4 ile birlikte JavaScript tabanlı yapılandırma (tailwind.config.js) yerini CSS-First yapılandırmaya bıraktı. Artık fontları CSS değişkenleri (Variables) üzerinden yönetiyoruz.

### 3.1. Font Ailesini Tanımlama ve Ezme

Tailwind'in varsayılan sans ailesini (Inter, Roboto vs. içeren o uzun listeyi) kendi fontumuzla güncellemek istiyoruz.

src/index.css dosyanızı açın:

```css
@import "tailwindcss";
@import "./assets/fonts/fonts.css"; /* Manuel fontlarımızı dahil ettik */

@theme {
  /* 1. Tailwind'in varsayılan "font-sans" utility'sini güncelliyoruz.
     En başa kendi fontumuzu (Axiforma veya Inter) yazıyoruz.
     Devamına "System Stack" ekliyoruz ki font yüklenmezse site bozulmasın.
  */
  --font-sans: "Axiforma", "Inter", ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji";
  
  /*
     2. Eğer fontsource kullanıyorsanız 'Axiforma' yerine
     'Inter Variable' veya 'Inter' yazmanız yeterlidir.
  */
}
```

### 3.2. Özel Ağırlıkları (Custom Weights) Yönetmek

Standart Tailwind sınıfları font-thin (100) ile font-black (900) arasındadır. Ancak Axiforma örneğindeki gibi Book (350) veya Heavy (850) gibi ara değerleriniz varsa bunları sisteme tanıtmanız gerekir.

Yine src/index.css içerisindeki @theme bloğuna ekleme yapıyoruz:

```css
@theme {
  /* ...font-sans tanımı... */

  /* Yeni Utility Classlar Üretiyoruz */
  --font-weight-book: 350;
  --font-weight-heavy: 850;
}
```

**Sonuç:** Tailwind bu tanımları gördüğünde otomatik olarak şu sınıfları oluşturur:

- `.font-book` → `font-weight: 350;`
- `.font-heavy` → `font-weight: 850;`

### 3.3. Kullanım Örneği

```javascript
export default function HeroSection() {
  return (
    <div className="font-sans"> {/* Otomatik Axiforma oldu */}
      
      {/* 850 Ağırlığında Başlık */}
      <h1 className="font-heavy text-4xl">
        Veteriner Kliniği Yönetimi
      </h1>
      
      {/* 350 Ağırlığında Alt Metin */}
      <p className="font-book text-gray-600 mt-2">
        Modern ve güvenilir çözüm ortağınız.
      </p>

      {/* Standart 700 Ağırlığında Buton */}
      <button className="font-bold bg-blue-500 text-white px-4 py-2">
        Giriş Yap
      </button>
    </div>
  )
}
```

## BÖLÜM 4: Performans İpuçları (Best Practices)

Fontları projeye eklemek yetmez, hızlı yüklenmelerini sağlamak gerekir.

### 4.1. Preloading (Ön Yükleme)

Tarayıcı, CSS'i okuyup "Bu sayfada Bold fonta ihtiyaç var" diyene kadar fontu indirmez. Bu gecikmeyi önlemek için, sadece ekranın ilk açılışında görünen (Above the fold) ana fontları index.html içinde preload edin.

```html
<link rel="preload" href="/src/assets/fonts/Axiforma/Axiforma-Regular.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/src/assets/fonts/Axiforma/Axiforma-Bold.woff2" as="font" type="font/woff2" crossorigin>
```

### 4.2. font-display: swap

Bu özellik Core Web Vitals (LCP ve CLS) skorları için kritiktir.

- **Swap:** Font yüklenene kadar sistem fontunu gösterir, yüklenince değiştirir. Yazı her zaman okunabilir kalır.
- **Block:** Font yüklenene kadar yazıyı gizler. (Önerilmez, kullanıcı boş sayfa görür).

### 4.3. Fallback Font Uyumu

Eğer özel fontunuz (Axiforma) ile sistem fontu (Arial/Sans-serif) arasında ciddi boyut farkı varsa, font yüklendiğinde sayfa zıplama yapar (Layout Shift). Bunu engellemek için size-adjust CSS özelliği kullanılabilir ancak bu ileri seviye bir konudur. Genellikle Tailwind config içindeki fallback listesini (ui-sans-serif, system-ui...) doğru sıralamak yeterlidir.

## Özet Karşılaştırma

| Özellik | @fontsource (Inter vb.) | Manuel Self-Hosted (Axiforma vb.) |
|---------|------------------------|----------------------------------|
| Kurulum | Çok Kolay (npm i) | Orta (Dosya yönetimi gerekir) |
| Performans | Mükemmel (Otomatik optimize) | Mükemmel (Doğru yapılandırılırsa) |
| Esneklik | Sadece kütüphanedeki fontlar | Her türlü font |
| Tailwind v4 | CSS değişkeni ile isim atanır | CSS değişkeni + Özel ağırlıklar |
| Dosya Boyutu | Variable font ile çok düşük | WOFF2 dönüşümü şart |

> **İpucu:** Font yönetimi performans ve kullanıcı deneyimi açısından kritiktir. Her zaman WOFF2 formatını kullanın ve font-display: swap özelliğini unutmayın.