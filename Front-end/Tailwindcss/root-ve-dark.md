## :root nedir?
:root, CSS'in kendisine ait özel bir sözde sınıftır (pseudo-class). HTML sayfasındaki en üstteki öğeyi (genellikle `<html>` öğesi) temsil eder.

Değişken tanımlamak için "Global Scope" burasıdır. Buraya yazdığın bir değişken örneğin --color-primary, sayfanın en dibindeki bir butondan bile erişilebilir.

```css
:root {
  /* Tüm sitede geçerli varsayılan değerler buraya yazılır */
  --color-bg: #ffffff;      /* Beyaz arka plan */
  --color-text: #000000;    /* Siyah yazı */
}
```

## .dark nedir?
.dark teknik olarak CSS için özel bir kelime değildir, sadece bir sınıf ismidir (class name). Ancak Tailwind CSS dünyasında "Karanlık Mod"u tetiklemek için kullanılan standart bir anahtar kelimedir.

Tailwind class="dark" ifadesini HTML etiketinde gördüğü anda dark: ile başlayan stilleri örneğin dark:bg-black devreye sokar.

HTML etiketine JavaScript ile dark sınıfını ekleyip çıkartarak kullanıcıya karanlık mod deneyimi sunabilirsin.

```css
.dark {
  /* HTML etiketinde 'dark' sınıfı varsa burası çalışır */
  --color-bg: #0f172a;      /* Koyu lacivert arka plan */
  --color-text: #ffffff;    /* Beyaz yazı */
}
```

## :root ve .dark birlikte nasıl çalışır
Tailwind v4'te tema yönetimi genellikle bu ikisinin kombinasyonuyla yapılır. Renk kodlarını doğrudan yazmak yerine değişkenleri değiştiririz. Bu sayede dark:text-white, dark:bg-black gibi her yere tek tek dark: yazmak zorunda kalmayız. Değişkenin değeri değişir, site otomatik dönüşür.

```css
@import "tailwindcss";

@theme {
  /* Tailwind'e bu değişkenleri tanıttık */
  --color-page-bg: var(--bg);
  --color-page-text: var(--text);
}

/* 1. Varsayılan (Aydınlık) Mod */
:root {
  --bg: #ffffff;
  --text: #333333;
}

/* 2. Karanlık Mod (Kullanıcı dark mode açtığında) */
/* .dark sınıfı html etiketine eklendiğinde bu değerler üsttekini ezer */
.dark {
  --bg: #0f172a; 
  --text: #f8fafc;
}
```
```html
<body class="bg-page-bg text-page-text">
  Merhaba Dünya
</body>
```

## Özet
:root: Değişkenlerin ilk tanımlandığı varsayılan yerdir. CSS standardıdır.
.dark: Karanlım mod sınıfıdır. Değişkenlerin koyu tema için değiştirildiği (override edildiği) yerdir.