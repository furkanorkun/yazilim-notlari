## Base Layer (Temel Katman)
Bu katman, tarayıcılar arasındaki stil farklarını sıfırlayan (Preflight/Reset) ve projenin genelindeki çıplak HTML etiketlerine (class'sız elementlere) varsayılan stiller atanılan yerdir.

Ne için kullanılır? <h1>, <p>, <body>, <a> gibi etiketlere global varsayılanlar atamak için.

```css
@layer base {
  body {
    font-family: var(--font-sans);
    background-color: var(--color-bg);
    color: var(--color-text);
  }

  h1 {
    @apply text-4xl font-bold;
  }

  a {
    text-decoration: none;
    @apply text-primary hover:underline;
  }
}
```

## Components Layer (Bileşen Katmanı)
Bu katman, birden fazla utility sınıfını bir araya getirerek oluşturduğun "soyutlanmış" sınıflar (örneğin .btn, .card) içindir.

Ne için kullanılır? Eğer React/Vue gibi component bazlı bir framework kullanmıyorsan (veya HTML içinde çok tekrar eden yapılar varsa) karmaşık stilleri tek bir class altına toplamak için.

React Geliştirici Notu: Sen React kullandığın için bu katmanı çok sık kullanman önerilmez. Buton stilini .btn class'ı oluşturmak yerine, bir <Button /> React bileşeni oluşturup utility class'ları (px-4 py-2 bg-blue-500) doğrudan JSX içinde kullanmak daha modern bir yöntemdir.

```css
@import "tailwindcss";

@layer components {
  .btn-primary {
    @apply px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition;
  }
  
  .card {
    @apply p-6 bg-white shadow-md rounded-xl border border-gray-200;
  }
}
```

## Utilities Layer (Yardımcı Katman)
Tailwind'in kalbi burasıdır. Tek bir işi yapan atomik sınıflar (örneğin flex, text-center, mt-4) bu katmanda yaşar. Bu katman en son yüklenir, böylece base veya components katmanındaki stilleri her zaman ezebilir (override edebilir).

Ne için kullanılır? Özel bir CSS özelliği eklemek istediğinde (örneğin standart Tailwind setinde olmayan bir text-shadow veya özel bir scrollbar stili) ve bunun hover:, md: gibi modifier'larla çalışmasını istediğinde.

```css
@import "tailwindcss";

/* v4'e özel yeni sözdizimi: Custom Utility Tanımlama */
@utility text-shadow-sm {
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
}

/* Artık HTML'de "hover:text-shadow-sm" şeklinde kullanabilirsin */
@utility scrollbar-thin {
  scrollbar-width: thin;
  scrollbar-color: var(--color-primary) var(--color-bg);
}
/* Artık HTML'de "md:scrollbar-thin" şeklinde kullanabilirsin */
```