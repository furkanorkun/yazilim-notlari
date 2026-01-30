# react-dropzone Nedir?
`react-dropzone`, uygulamanıza dosya sürükleyip bırakma alanı eklemenizi sağlayan bir react hook'udur. Temel olaraki kullanıcı dosyayı bir alana bıraktığında veya tıkladığında size o dosyanın ham verisini (File Object) verir.

Neden tercih edilir?
- Dosya türü (MIME type) ve boyut gibi kısıtlamaları kolayca uygulayabilirsiniz.
- Görünümü (UI) tamamen size bırakır. (Headless yapı)
- Klavye navigasyonu (Fare kullanmadan TAB tuşu ile focus. Normalde sıradan bir div elemanına TAB tuşu ile focus olamazsınız. Bu yüzden div'e tabIndex="0" otomatik eklenir. Focus olduktan sonra ENTER veya SPACE ile eylem tetiklenebilir.) ve ekran okuyucuları otomatik destekler (Arka planda `<input type="file" />` otomatik eklenir).

**İÇİNDEKİLER**
- [react-dropzone Nedir?](#react-dropzone-nedir)
  - [Kurulum](#kurulum)
  - [Temel Kullanım (useDropzone Hook)](#temel-kullanım-usedropzone-hook)
  - [Önemli Property'ler ve Yapılandırma](#önemli-propertyler-ve-yapılandırma)
  - [Kritik Özellikler ve İpuçları](#kritik-özellikler-ve-i̇puçları)
    - [A. Dosya Türü Kısıtlama (accept)](#a-dosya-türü-kısıtlama-accept)
    - [B. Görsel Önizleme (Preview) Oluşturma](#b-görsel-önizleme-preview-oluşturma)
    - [C. Hata Yönetimi (fileRejections)](#c-hata-yönetimi-filerejections)
  - [Özet](#özet)
  - [Ek notlar](#ek-notlar)
    - [A. URL.createObjectURL(file) Nedir?](#a-urlcreateobjecturlfile-nedir)
    - [B. Kod Bloğu Analizi](#b-kod-bloğu-analizi)

## Kurulum
Projenize eklemek için terminalde şu komutu çalıştırın:

```bash
npm install react-dropzone
```

## Temel Kullanım (useDropzone Hook)
Modern React'te en temiz kullanım şekli useDropzone hook'unu kullanmaktır.

İşte en basit haliyle çalışan bir örnek:
```jsx
import React, { useCallback } from 'react';
import { useDropzone } from 'react-dropzone';

function DosyaYuklemeAlani() {
  const onDrop = useCallback(acceptedFiles => {
    // Dosyalar buraya düşer
    console.log(acceptedFiles);
    // Burada dosyaları sunucuya gönderme işlemini başlatabilirsiniz.
  }, []);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({ onDrop });

  return (
    <div {...getRootProps()} style={styles.dropzone}>
      <input {...getInputProps()} />
      {
        isDragActive ?
          <p>Dosyaları buraya bırakın...</p> :
          <p>Dosyaları buraya sürükleyin veya seçmek için tıklayın.</p>
      }
    </div>
  );
}

const styles = {
  dropzone: {
    border: '2px dashed #cccccc',
    padding: '20px',
    textAlign: 'center',
    cursor: 'pointer'
  }
};

export default DosyaYuklemeAlani;
```

**Kodun Mantığı:**
1. `getRootProps:` Dropzone'un kapsayıcı (wrapper) div'ine yayılır (spread edilir). Bu, sürükleme olaylarını ve odaklanmayı yönetir.
2. `getInputProps:` Gizli bir `<input type="file" />` oluşturur. Bu, tıklandığında dosya gezgininin açılmasını sağlar.
3. `isDragActive:` Kullanıcı o anda dosyayı alanın üzerinde tutuyor mu? (Stil değiştirmek için harikadır).

## Önemli Property'ler ve Yapılandırma
useDropzone hook'una geçebileceğiniz en kritik ayarlar şunlardır:

| Property   | Açıklama ve Kullanım                                                                                                               |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `onDrop`   | En önemli fonksiyondur. Kullanıcı dosyaları bıraktığında çalışır. (acceptedFiles, fileRejections) => void şeklinde parametre alır. |
| `accept`   | Hangi dosya türlerine izin verildiğini belirler. (MIME types). Örnek: `{'image/*'}` sadece resim dosyalarına izin verir.           |
| `maxFiles` | Yüklenebilecek maksimum dosya sayısını sınırlar.                                                                                   |
| `maxSize`  | Yüklenebilecek maksimum dosya boyutunu byte cinsinden sınırlar. Örnek: `5 * 1024 * 1024` 5MB sınırı koyar.                         |
| `minSize`  | Yüklenebilecek minimum dosya boyutunu byte cinsinden sınırlar. Neden kullanılır ki? :)                                             |
| `multiple` | true (varsayılan) ise çoklu dosya, false ise tek dosya seçimine izin verir. Örneğin: multiple: maxFiles > 1                        |
| `disabled` | Dropzone'u devre dışı bırakır.                                                                                                     |
| `noClick`  | true ise dropzone tıklanamaz hale gelir (sadece sürükle bırak desteklenir).                                                        |

## Kritik Özellikler ve İpuçları
### A. Dosya Türü Kısıtlama (accept)
React-dropzone v14+ sürümünde accept prop'u özel bir nesne yapısı kullanır. Doğru Kullanım: Sadece Resim (PNG ve JPEG) ve PDF kabul etmek istiyorsanız:

```jsx
const { getRootProps, getInputProps } = useDropzone({
  accept: {
    'image/png': ['.png'],
    'image/jpeg': ['.jpg', '.jpeg'],
    'application/pdf': ['.pdf']
  }
});
```

### B. Görsel Önizleme (Preview) Oluşturma
Kullanıcı bir resim yüklediğinde, sunucuya göndermeden önce tarayıcıda göstermek harika bir UX (Kullanıcı Deneyimi) sağlar.
```jsx
import React, { useState, useCallback, useEffect } from 'react';
import { useDropzone } from 'react-dropzone';

function ResimOnizleme() {
  const [files, setFiles] = useState([]);

  const onDrop = useCallback(acceptedFiles => {
    setFiles(acceptedFiles.map(file => Object.assign(file, {
      preview: URL.createObjectURL(file) // Tarayıcı belleğinde geçici URL oluştur
    })));
  }, []);

  const { getRootProps, getInputProps } = useDropzone({
    onDrop,
    accept: {'image/*': []} // Tüm resim türleri
  });

  // Bellek sızıntısını önlemek için URL'leri temizle
  useEffect(() => {
    return () => files.forEach(file => URL.revokeObjectURL(file.preview));
  }, [files]);

  return (
    <section>
      <div {...getRootProps({className: 'dropzone'})}>
        <input {...getInputProps()} />
        <p>Resimleri buraya sürükleyin</p>
      </div>
      <aside>
        {files.map(file => (
          <div key={file.name}>
             <img src={file.preview} style={{width: '100px'}} alt="preview" />
          </div>
        ))}
      </aside>
    </section>
  );
}
export default ResimOnizleme;
```

### C. Hata Yönetimi (fileRejections)
Eğer kullanıcı yanlış dosya türü veya boyutu yüklerse, fileRejections parametresini kullanarak bunu kullanıcıya bildirebilirsiniz.

```jsx
const { getRootProps, getInputProps, fileRejections } = useDropzone({
  accept: {'image/jpeg': []},
  maxSize: 1024 * 1024 // 1MB
});

const fileRejectionItems = fileRejections.map(({ file, errors }) => (
  <li key={file.path}>
    {file.path} - {file.size} bytes
    <ul>
      {errors.map(e => <li key={e.code}>{e.message}</li>)}
    </ul>
  </li>
));
```

## Özet
* React-dropzone, dosya yükleme arayüzünü (UI) size bırakır, arkadaki mantığı (logic) halleder.
* Hook yapısı (useDropzone) en modern ve temiz kullanım şeklidir.
* accept prop'unu kullanırken MIME type nesne yapısına dikkat edin.
* Bu kütüphane dosyayı sunucuya yüklemez; sadece dosyayı size verir. Yükleme işlemini (örneğin Axios kullanarak) onDrop içinde sizin yapmanız gerekir.

## Ek notlar
### A. URL.createObjectURL(file) Nedir?
Bu, tarayıcının (Browser) sunduğu yerel bir JavaScript fonksiyonudur.

* **Sorun**: Bir `<img src="...">` etiketi, kaynağı olarak bir metin (URL) bekler (örneğin: https://site.com/resim.jpg). Ancak elinizdeki File nesnesi (kullanıcının bilgisayarından seçtiği dosya) bir metin değil, bir veri yığınıdır (binary data).
* **Çözüm**: URL.createObjectURL(file), bilgisayarın belleğindeki (RAM) bu dosyaya işaret eden sahte, geçici bir URL (link) üretir.

**Örnek Çıktı**: Bu fonksiyonu çalıştırdığınızda size şöyle bir string verir: "blob:http://localhost:3000/1234-5678-abcd-efgh"

Nasıl Çalışır? Tarayıcıya şunu dersiniz: "Bu dosyayı bellekte tut ve bana ona ulaşabileceğim kısa bir adres ver." Artık bu adresi `<img src={...} />` içine koyduğunuzda resim görüntülenir.

### B. Kod Bloğu Analizi
```jsx
setFiles(acceptedFiles.map(file => Object.assign(file, {
  preview: URL.createObjectURL(file)
})));
```

Bu kodun amacı: **"Dropzone'dan gelen ham dosya nesnelerini al, her birinin içine 'preview' adında bir özellik ekle ve bu özelliğe geçici resim linkini koy."**

Adım adım ne olduğunu görelim:

**Adım A:** acceptedFiles.map(...)
acceptedFiles, kullanıcının o an sürüklediği dosyaların listesidir (bir Array). map fonksiyonu bu listedeki her bir dosya için tek tek döngüye girer.

**Adım B:** URL.createObjectURL(file)
Döngüdeki o anki dosya için geçici bir link oluşturulur (örn: blob:...).

**Adım C:** `Object.assign(file, { preview: ... })`
Burası işin kilit noktasıdır.

* Normalde file nesnesinin içinde sadece name, size, type gibi özellikler vardır.
* Object.assign, mevcut file nesnesini alır ve içine yeni bir özellik (property) enjekte eder.
* Bizim durumumuzda, dosya nesnesine preview isminde yeni bir alan ekliyoruz ve değerini az önce oluşturduğumuz blob URL'i yapıyoruz.

Dönüşüm Şöyle Olur:

**Önce (Ham Dosya):**
```jsx
{
  name: "profil.jpg",
  size: 1024,
  type: "image/jpeg"
}
```

**Sonra (İşlenmiş Dosya):**
```jsx
{
  name: "profil.jpg",
  size: 1024,
  type: "image/jpeg",
  preview: "blob:http://localhost:3000/1234-5678-abcd-efgh" // <-- Yeni eklenen kısım
}
```

**Adım D: setFiles(...)**
Son olarak, bu modifiye edilmiş (içinde preview linki olan) dosya listesi React state'ine (files) kaydedilir.

**Neden Böyle Yapıyoruz?**
Bunu yapmamızın sebebi, JSX (HTML) kısmında resmi göstermek istediğimizde kolayca erişebilmektir:

```jsx
// Kodun devamında (JSX içinde)
{files.map(file => (
  <div key={file.name}>
     {/* Artık file.preview diyerek resme ulaşabiliriz! */}
     <img src={file.preview} alt="Önizleme" />
  </div>
))}
```

**Önemli Bir Uyarı (Bellek Temizliği)**
`URL.createObjectURL` ile oluşturulan linkler, tarayıcı sekmesi kapanana kadar bellekte (RAM) yer kaplar. Eğer çok fazla resim yüklenirse bilgisayar yavaşlayabilir. Bu yüzden React bileşeni ekrandan gittiğinde (unmount) bu linkleri imha etmek iyi bir pratiktir:

```jsx
useEffect(() => {
  // Temizlik fonksiyonu
  return () => files.forEach(file => URL.revokeObjectURL(file.preview));
}, [files]);
```