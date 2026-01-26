# 1. Veri tanımlama ve Erişim
Bir tablonun veriyi nasıl okuyacağı, işleyeceği ve kimliklendireceği ColumnDef içerisindeki üç temel özellik ile belirlenir: accessorKey, accessorFn ve id.

## 1.1. Accessor stratejileri

### 1. accessorKey (Doğrudan erişim)
Veritabanından gelen obje içerisindeki bir anahtara (key) doğrudan karşılık gelen sütunlar için kullanılır.
* İşlevi: Veriyi okur ve sütuna otomatik olarak o key'in adını ID olarak atar.
* Kullanım: Veri üzerinde işlem yapılmadan direkt olarak gösterilmesi gereken sütunlarda tercih edilir.
* Örnek: email, userName, status gibi

```ts
{
  accessorKey: "email", // Hem veri kaynağı hem de columnId "email" olur.
  header: "E-Posta",
}
```

### 2. accessorFn (Hesaplanmış erişim)
Verinin ham haliyle değil, işlenmiş, birleştirilmiş veya dönüştürülmüş haliyle tabloya sunulması gerektiğinde kullanılır.
* İşlevi: Dışarıdan bir fonksiyon alır ve bu fonksiyon satır verisini işleyerek sütun için gösterilecek değeri üretir.
* Kullanım: İki veya daha fazla alanın birleştirilmesi, tarih formatlama, özel hesaplamalar gibi durumlarda tercih edilir.
* Örnek: fullName (firstName + lastName), formattedDate gibi

Not: accessorFn kullanıldığında, kolonun benzersiz bir şekilde tanımlanabilmesi için id özelliğinin de belirtilmesi gerekir.
Not: Sıralama ve filtreleme işlemleri, accessorFn tarafından döndürülen değerlere göre yapılır.

```ts
{
  id: "fullName", // accessorKey olmadığı için ID zorunludur.
  accessorFn: (row) => `${row.firstName} ${row.lastName}`, // Sıralama bu string'e göre yapılır.
  header: "Tam İsim"
}
```

### 3. id (Manuel kimliklendirme)
Sütunun benzersiz kimliğidir. accessorKey kullanıldığında otomatik oluşur, ancak diğer durumlarda manuel verilmelidir. 
* İşlevi: Sütunun benzersiz kimliğini belirler.
* Kullanım: Veri çekmeyen sütunlar (Actions, Select) veya accessorFn kullanılan sütunlar.
* Örnek: actions, customColumn gibi

```ts
{
  id: "actions", // Veri kaynağı yok, manuel ID gerekli.
  cell: ({ row }) => <ActionMenu user={row.original} />, // row.original üzerinden ham veriye erişilir.
  header: "İşlemler"
}
```

Karşılaştırma Tablosu:
| Özellik       | accessorKey          | accessorFn + id           | Sadece id       |
|---------------|----------------------|---------------------------|-----------------|
| Veri Kaynağı  | Obje Key'i (String)  | Fonksiyon Return Değeri   | Yok (UI Odaklı) |
| ID Ataması    | Otomatik             | Manuel                    | Manuel          |
| Kullanım Yeri | email, age           | fullName, statusText      | actions, select |

# 2. Satır Verisine Erişim Yöntemleri

## 2.1. row.original (Ham veri)
Veritabanından gelen TData tipindeki objenin, hiçbir değişikliğe uğramamış halidir.

* Kullanım Alanı: API istekleri (Update/Delete), Modal açma, ID erişimi.
* Avantajı: Tabloda gösterilmeyen gizli verilere (örn: secretToken, db_id) erişim sağlar. Tip güvenliği tamdır.

## 2.2. row.getValue("columnId") (İşlenmiş veri)
TanStack Table motoru tarafından işlenmiş, accessorKey veya accessorFn sonucunda dönen değerdir.

* Kullanım Alanı: Hücre içi görselleştirme, koşullu render.
* Kritik Not: Sıralama ve filtreleme bu değer üzerinden yapılır.

## Önemli Not: Data vs UI Ayrımı
Sıralama (sorting) mantığının bozulmaması için veri dönüşümü ile görsel formatlama ayrılmalıdır.
Bunu çok basit bir bakkal defteri örneğiyle açıklayalım. Elimizde şu veri olsun: 
- Elma: 25 TL
- Bilgisayar: 1000 TL
- Sakız: 5 TL

1. Hatalı Yaklaşım (Formatlama accessorFn içinde)
Eğer accessorFn içinde "TL" eklersen, TanStack Table bu sayıları artık SAYI (Number) olarak değil, YAZI (String) olarak görür. Bilgisayarlar yazıları alfabetik (sözlük) sırasına göre dizer. Sözlükte "A" harfi "B"den önce gelir. Sayısal stringlerde de durum aynıdır; sadece ilk karaktere bakar:
- "1000 TL" (İlk karakteri '1') -> En başa gelir.
- "25 TL" (İlk karakteri '2')
- "5 TL" (İlk karakteri '5') -> En sona gelir.
Sonuç (Sıralama Bozuk): 
- 1000 TL (En ucuz buymuş gibi davranır çünkü 1, 5'ten küçüktür) 
- 25 TL
- 5 TL 

Kod Örneği:

```JavaScript
// YANLIŞ KULLANIM
{
  header: 'Fiyat',
  // HATA: Veriyi burada string'e çevirdin. Tablo artık sayı olduğunu bilmiyor.
  accessorFn: (row) => `${row.price} TL`, 
}
```

2. Doğru Yaklaşım (Ayrım Yapmak)
Burada görevi ikiye bölüyoruz: 
* Accessor (Mantık): Tabloya sadece ham sayıyı veriyoruz ki sıralamayı matematiksel yapsın ($5 < 25 < 1000$).
* Cell (Makyaj/Görsel): Kullanıcıya gösterirken sonuna "TL" ekliyoruz.

Sonuç (Sıralama Doğru): Tablo arka planda şuna bakar: 5, 25, 1000. Sıralama doğru çalışır. Ama ekrana yazarken senin cell fonksiyonunu kullanır ve yanlarına TL yazar.

Kod Örneği:

```JavaScript
// DOĞRU KULLANIM
{
  header: 'Fiyat',
  // 1. ADIM: Tabloya ham sayıyı ver (Sıralama için bunu kullanır)
  accessorFn: (row) => row.price, 

  // 2. ADIM: Ekrana basarken süsle (Sadece görüntü içindir)
  cell: (info) => {
    const hamDeger = info.getValue(); // 1000
    return `${hamDeger} TL`;          // "1000 TL" olarak ekrana basar
  }
}
```
Özetle 
* Accessor: Tablonun beynidir (Hesaplama, sıralama, filtreleme buradaki veriye göre yapılır). Buraya Number girmelisin.
* Cell: Tablonun yüzüdür (Sadece son kullanıcı ne görsün istiyorsan o yapılır). Buraya String (süslü yazı) girebilirsin.

# 3. Görselleştirme Katmanları
Her sütun tanımı (ColumnDef), üç farklı render noktası sunar: header, cell ve footer.

## 3.1. Header (header)
Sütun başlığını temsil eder. Basit bir string olabileceği gibi, sıralama ve filtreleme kontrollerini içeren karmaşık bir bileşen de olabilir.

Örnek: Sıralama ve Aksiyon Menülü Gelişmiş Header
```JavaScript
{
  accessorKey: "role",
  header: ({ column }) => (
    <DataTableColumnHeader column={column} title="Kullanıcı Rolü" />
  ),
}
```

Not: DataTableColumnHeader bileşeni, column.toggleSorting() ve column.toggleVisibility() metodlarını kullanarak tablo state'ini yönetir.

## 3.2. Cell (cell)
Satır hücresinin içeriğidir. Veriyi son kullanıcıya sunma katmanıdır.

Senaryo A: Formatlama
```tsx
cell: ({ row }) => {
  const amount = parseFloat(row.getValue("amount"));
  return new Intl.NumberFormat("tr-TR", { style: "currency", currency: "TRY" }).format(amount);
}
```

Senaryo B: Aksiyonlar (Action Column)
```tsx
{
  id: "actions",
  cell: ({ row }) => {
    const user = row.original; // API işlemleri için ham veri şart
    return (
      <Button onClick={() => deleteUser(user.id)}>Sil</Button>
    );
  }
}
```

## 3.3. Footer (footer)
Sütun özetleri (toplam, ortalama) için kullanılır. UI'da görünmesi için <tfoot> elementinin render edilmesi gerekir.

```tsx
footer: ({ table }) => `Toplam ${table.getFilteredRowModel().rows.length} kayıt`
```

# 4. Fonksiyonel Özellikler ve Konfigürasyon

ColumnDef içerisinde sütun davranışlarını kontrol eden temel bayraklar (flags) ve fonksiyonlar şunlardır:

1. Sıralama (Sorting)
* enableSorting: boolean. Sütunun sıralanabilirliğini açar/kapatır.
* sortingFn: Varsayılan sıralama algoritmasını (alphanumeric) ezer. Özel veri tipleri için gereklidir.
* invertSorting: Sıralama yönünü (ASC/DESC) tersine çevirir.

2. Filtreleme (Filtering)
* enableColumnFilter: boolean. Sütun bazlı filtrelemeyi açar.
* filterFn: Filtreleme mantığını belirler (örn: includesString, equals, arrIncludesSome).
* enableGlobalFilter: Global arama kutusunun bu sütunu tarayıp taramayacağını belirler.

3. Görünüm ve Boyutlandırma
* enableHiding: Kullanıcının sütunu gizleyip gizleyemeyeceğini belirler. (Actions sütunu için genelde false yapılır).
* size, minSize, maxSize: Piksel cinsinden genişlik tanımları. enableResizing: true olduğunda sınırları belirler.

4. Meta Veri (meta)
Tablo kütüphanesinin varsayılan olarak desteklemediği ama sizin UI katmanınızda ihtiyaç duyduğunuz özel verileri taşır.

```tsx
// Tanım
meta: {
  align: "right",
  className: "text-red-500 font-bold"
}

// Kullanım (Generic Table Component içinde)
<td className={column.columnDef.meta?.className}>
  ...
</td>
```

# 5. İleri Seviye: Kolon Gruplama (Column Grouping)
Birden fazla sütunu ortak bir başlık altında toplamak için columns özelliği kullanılır. Bu yapı iç içe (nested) olabilir.

```tsx
const columns: ColumnDef<User>[] = [
  {
    header: 'Kişisel Bilgiler',
    columns: [ // Alt sütunlar
      { accessorKey: 'firstName', header: 'Ad' },
      { accessorKey: 'lastName', header: 'Soyad' },
    ],
  },
  {
    header: 'İletişim',
    columns: [
      { accessorKey: 'email', header: 'E-Posta' },
      { accessorKey: 'phone', header: 'Telefon' },
    ],
  },
];
```