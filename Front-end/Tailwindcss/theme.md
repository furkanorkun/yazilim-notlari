# Tailwind CSS Theme Sistemi

Tailwind CSS'te **theme**, projenizin görsel kimliğini oluşturan tasarım sisteminin merkezi konfigürasyonudur. Kısaca; renk paleti, font, spacing, breakpoint, border-radius gibi tüm tasarım değişkenlerinin tanımlandığı yerdir. Örneğin `p-4`, `text-blue-500` gibi Tailwind sınıfları oluştururken theme ayarları referans alır.

## Theme Neleri Kontrol Eder?

Tailwind'in varsayılan olarak gelen bir teması vardır. Ancak theme ayarları ile şunları özelleştirebilirsiniz:

- **Colors:** `bg-red-500`, `text-primary` gibi sınıfların renk kodları
- **Spacing:** `p-4`, `m-2`, `w-96` gibi sınıfların piksel/rem karşılıkları
- **Screens (Breakpoints):** `md:`, `lg:` gibi responsive öneklerinin hangi pikselde devreye gireceği
- **Typography:** `font-sans`, `text-xl`, `leading-tight` gibi font ailesi ve boyut ayarları
- **Border & Effects:** `rounded-lg`, `shadow-xl`, `opacity-50` değerleri

## Theme Ayarları Nasıl Yapılır?

**Tailwind v4** ile JavaScript konfigürasyonları yerine native CSS değişkenleri kullanılır. Artık `tailwind.config.js` dosyasına bağımlılık azalmıştır ve tema ayarları doğrudan CSS dosyasında yapılır. Bu yeni sistemde `extend` mantığı varsayılan davranıştır. (Önceden extend ile ekleme yapılmazsa tüm tema geçersiz sayılırdı, sadece eklenen kullanılabilirdi.)

```css
@import "tailwindcss";

@theme {
  /* Renk Tanımlama */
  --color-primary: #3b82f6;
  --color-danger: #ef4444;

  /* Font Tanımlama */
  --font-display: "Satoshi", sans-serif;

  /* Breakpoint Tanımlama */
  --breakpoint-3xl: 1920px;

  /* Spacing (Boşluk) Tanımlama */
  --spacing-128: 32rem;
}
```

## Nasıl Çalışır?

Yukarıdaki gibi `--color-primary` tanımladığınızda, Tailwind otomatik olarak `text-primary`, `bg-primary`, `border-primary` sınıflarını oluşturur.

### Avantajları:
- JavaScript dosyası ile uğraşmadan, doğrudan CSS içinde tarayıcının anlayacağı değişkenlerle çalışırsınız
- Daha hızlı derleme ve daha az konfigürasyon dosyası
- CSS değişkenleri sayesinde runtime'da tema değiştirme imkanı

## Örnek Kullanım Tablosu

| Theme Değişkeni | Oluşturulan Sınıflar | Açıklama |
|-----------------|---------------------|----------|
| `--color-primary: #3b82f6` | `text-primary`, `bg-primary`, `border-primary` | Ana renk paleti |
| `--font-display: "Satoshi"` | `font-display` | Özel font ailesi |
| `--breakpoint-3xl: 1920px` | `3xl:` | Büyük ekran breakpoint'i |
| `--spacing-128: 32rem` | `p-128`, `m-128` | Özel boşluk değeri |

> **İpucu:** Theme değişkenlerinizi mantıklı bir şekilde adlandırın. Örneğin `--color-brand` yerine `--color-primary` kullanın ki Tailwind'in otomatik sınıf oluşturma mantığı ile uyumlu olsun.