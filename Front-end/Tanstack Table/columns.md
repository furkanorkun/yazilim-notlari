# TanStack Table'da row.original vs row.getValue() Farkı

TanStack Table (React Table v8) kullanırken bu ikisi arasındaki farkı anlamak çok kritiktir çünkü veriye erişim yöntemini belirler.

## Kısaca

- **row.original**: Veritabanından gelen ham (raw) veri objesidir.
- **row.getValue("key")**: TanStack Table tarafından işlenmiş, o sütuna ait değerdir.

## Detaylı Farklar

### 1. row.original (Ham Veri)

Bu, tabloya data prop'u olarak verdiğin dizideki o satıra denk gelen objenin ta kendisidir. Hiçbir değişikliğe uğramamış halidir.

**Ne zaman kullanılır?** Sütunda gösterilmeyen ama o satırla ilgili bir işlem yaparken (örneğin ID'ye ihtiyaç duyduğunda) veya tüm objeyi bir yere (modal vb.) göndermen gerektiğinde.

**Örnek Senaryo:** Verin şu şekilde olsun:

```typescript
const user = {
  id: 101,
  firstName: "Ahmet",
  lastName: "Yılmaz",
  secretToken: "xyz-123" // Ekranda göstermiyoruz ama lazım
}
```

Tablonda "İşlemler" sütunu yapıyorsun:

```typescript
{
  id: "actions",
  cell: ({ row }) => {
    // 1. row.original ile TÜM veriye erişirsin
    const user = row.original

    console.log(user.secretToken) // "xyz-123" (Erişebilirsin)

    return (
      <button onClick={() => console.log("Edit ID:", user.id)}>
        Düzenle
      </button>
    )
  }
}
```

### 2. row.getValue("columnId") (İşlenmiş Veri)

Bu fonksiyon, ilgili sütunun accessorKey veya accessorFn ayarına göre hesapladığı/döndürdüğü değeri verir.

**Ne zaman kullanılır?** Sütunun değerini formatlarken, sıralama mantığında veya o hücrenin o anki hesaplanmış değerine ihtiyacın olduğunda.

**Örnek Senaryo:** Ad ve Soyadı birleştiren bir sütunun var.

```typescript
{
  id: "fullName", // Sütun ID'si
  // Accessor function: Veriyi dönüştürüyor
  accessorFn: (row) => `${row.firstName} ${row.lastName}`,
  cell: ({ row }) => {

    // row.original dersen { firstName: "Ahmet", lastName: "Yılmaz" } alırsın.
    // "Ahmet Yılmaz" stringi original objede YOK.

    // row.getValue("fullName") dersen "Ahmet Yılmaz" alırsın.
    const fullName = row.getValue("fullName")

    return <b>{fullName}</b>
  }
}
```

## Karşılaştırma Tablosu

| Özellik          | row.original                  | row.getValue(columnId)              |
|------------------|-------------------------------|-------------------------------------|
| Döndürdüğü Şey  | Veri objesinin tamamı (TData). | O sütun için hesaplanmış tek bir değer. |
| Bağımlılık      | Sütun tanımlarından bağımsızdır. | accessorKey veya accessorFn'e bağlıdır. |
| Tip Güvenliği   | Tamamıyla tip güvenlidir (Typescript). | Genelde unknown dönebilir, cast etmen gerekebilir. |
| Görünmez Veri   | Tabloda olmayan verilere erişir. | Sadece tanımlı sütunlara erişir. |
| Kullanım Yeri   | Edit/Delete butonları, Modal açma. | Hücre içi formatlama, Sorting, Filtering. |

## E-Tablolar'a aktar

## Kritik Bir "Hata" Örneği

Diyelim ki tablonda amount (para) sütunu var ve bunu accessorFn ile string'e çevirdin ("100 TL").

```typescript
// Sütun tanımı
{
  accessorKey: "amount",
  // DİKKAT: Veriyi burada string'e çevirirsen sorting bozulabilir!
  // accessorFn: (row) => `${row.amount} TL`,
}

// Cell içinde kullanım
cell: ({ row }) => {
    // 1. row.original.amount -> 100 (Number) -> Matematiksel işlem yapabilirsin.
    const rawAmount = row.original.amount

    // 2. row.getValue("amount") -> Eğer accessorFn kullansaydın "100 TL" (String) gelirdi.
    const cellValue = row.getValue("amount")

    // Best Practice:
    // Accessor (getValue) sıralama/filtreleme için ham veri (number) döndürmeli,
    // Görselleştirme (TL ekleme) işi 'cell' render içinde yapılmalı.
}
```

## Özet: Hangisini Seçmeliyim?

- **Veriyi ekrana basmak (Render) için:** Genellikle row.getValue() (veya direkt info.getValue()) daha temizdir çünkü TanStack'in veri okuma mantığına sadık kalır.
- **API isteği atmak (Delete/Update) için:** Kesinlikle row.original kullanmalısın. Çünkü API senden "Ahmet Yılmaz" stringini değil, id: 101 bilgisini ister ve id muhtemelen original objenin içindedir.

## id ile accessorKey arasında ne fark var?

Bu ikisi birbirine çok karışır ama aralarındaki farkı anlamak, özellikle karmaşık tablolarda (sorting, hiding columns) hayat kurtarır.

### Kısaca özetlemek gerekirse:

- **accessorKey:** Hem veriyi çeker hem de kimlik (ID) verir. (Otomatik pilot).
- **id:** Sadece sütunun ismini/kimliğini belirler. Veri çekmez. (Manuel kontrol).

### Detaylarına bakalım:

#### 1. accessorKey (Sihirli Değnek)

Bu, veritabanındaki bir alana (property) doğrudan bağlanmak için kullanılır. TanStack Table'a şunu dersin: "Git, veri objemdeki email alanını al, sütun ID'sini de otomatik olarak email yap."

**Özellikleri:**

- Veriyi otomatik okur: row.original.email ile uğraşmana gerek kalmaz.
- ID'yi otomatik atar: Sen belirtmesen bile sütunun ID'si email olur.
- Sorting/Filtering otomatik çalışır: Tablo hangi alana bakacağını bildiği için sıralamayı otomatik yapar.

```typescript
// Veri: { email: "ali@test.com", age: 25 }

// Kullanım
{
  accessorKey: "email", // Hem veriyi çeker, hem ID 'email' olur
  header: "E-Posta",
}
```

#### 2. id (Kimlik Kartı)

Bir sütunun benzersiz adıdır. TanStack Table'ın sütunları birbirinden ayırt etmesi, sütun gizleme/gösterme (visibility) veya sıralama (sorting) state'lerini yönetmesi için her sütunun mutlaka bir ID'si olmalıdır.

Ancak id tek başına veri çekemez.

**Ne zaman id kullanmak ZORUNDASIN?**

- Veriyle ilgisi olmayan sütunlarda: Örn: "Actions" (Sil/Düzenle butonları), "Select" (Checkbox).
- Hesaplanmış (Computed) sütunlarda: accessorFn kullandığında tablo bu sütuna ne isim vereceğini bilemez, senin vermen gerekir.

**Karşılaştırmalı Örnekler**

**Senaryo 1: Basit Veri (accessorKey yeterli)**

Verimiz username olsun.

```typescript
{
  accessorKey: "username", // ID otomatik olarak "username" olur.
  header: "Kullanıcı Adı"
}
```

**Senaryo 2: Hesaplanmış Veri (accessorFn + id)**

Ad ve soyad birleşecek. Tablo bunun ID'sinin ne olduğunu bilemez (firstName mi? lastName mi?). id vermezsen hata alırsın.

```typescript
{
  id: "fullName", // ZORUNLU: Tablo bu sütunu "fullName" olarak tanıyacak
  accessorFn: (row) => `${row.firstName} ${row.lastName}`,
  header: "Tam İsim",
}
```

**Senaryo 3: İşlem Sütunu (Sadece id)**

Burada veri çekilmiyor, sadece buton var.

```typescript
{
  id: "actions", // ZORUNLU: State yönetimi için (örn: bu sütunu gizlemek istersen)
  header: "İşlemler",
  cell: ({ row }) => <button>Sil</button>
}
```

**Kritik Bir Detay: accessorKey varken id verilir mi?**

Evet, verilebilir ama genelde gerek yoktur. accessorKey kullanırsan tablo ID'yi zaten o key yapar. Ama bazen ID'yi değiştirmek istersin.

Örnek: Veritabanında alanın adı user_id_fk_123 gibi çirkin bir şeydir ama sen kodda temiz görünsün istersin.

```typescript
{
  accessorKey: "user_id_fk_123", // Veriyi buradan çek
  id: "userId", // Ama tablo state'inde (url'de vs.) adı "userId" olsun
  header: "ID"
}
```

### Özet Tablo

| Özellik       | accessorKey                  | id                                  |
|---------------|------------------------------|-------------------------------------|
| Görevi       | Veri çekmek + ID atamak     | Sütunu isimlendirmek                |
| Veri Kaynağı | Doğrudan obje key'i (string) | Yok (veya accessorFn ile gelir)     |
| Zorunluluk   | Opsiyonel                   | Her sütunda olmak zorunda (ama accessorKey varsa otomatik oluşur) |
| Kullanım Yeri| email, status, amount       | actions, select, fullname (hesaplanmış) |
| Tavsiye      | Veritabanındaki bir alana birebir karşılık geliyorsa her zaman accessorKey kullan. Karmaşık işlemler veya butonsa id kullan. |

## accessorFn nedir

accessorFn (Accessor Function), TanStack Table'a veriyi "nasıl okuyacağını" ve gerekirse okurken "nasıl dönüştüreceğini" söylediğin bir fonksiyondur.

accessorKey veritabanındaki bir anahtarı (key) doğrudan alırken, accessorFn veriyi alıp işledikten sonra tabloya verir.

Bunu bir "Dönüştürücü" (Transformer) gibi düşünebilirsin.

### Ne Zaman Kullanılır?

Genellikle şu 3 senaryoda accessorKey yetersiz kalır ve accessorFn kullanırsın:

#### 1. Verileri Birleştirmek (Merge)

Veritabanında Ad ve Soyad ayrıdır, ama sen tabloda tek sütunda "Tam İsim" göstermek ve buna göre sıralama yapmak istiyorsun.

```typescript
// Veri: { firstName: "Ahmet", lastName: "Yılmaz" }

{
  id: "fullName", // accessorKey olmadığı için ID vermek ZORUNLU
  header: "Ad Soyad",
  // row: Tüm satır verisidir
  accessorFn: (row) => `${row.firstName} ${row.lastName}`,
}
// Çıktı (Value): "Ahmet Yılmaz"
// Tablo artık "Ahmet Yılmaz" stringine göre sıralama (sort) yapar.
```

#### 2. Derin İç İçe (Nested) Veriler

Verin çok karmaşık bir obje yapısındaysa ve accessorKey ile nokta atışı (user.details.address.city) yapmak bazen tip güvenliği (TypeScript) açısından zorluyorsa veya arada bir kontrol (null check) gerekiyorsa kullanılır.

```typescript
// Veri: { company: { address: { city: "İstanbul" } } }

{
  id: "city",
  header: "Şehir",
  accessorFn: (row) => row.company?.address?.city || "Bilinmiyor",
}
```

#### 3. Veri Tipi Dönüştürme (Logic)

Veri veritabanında 0 veya 1 olarak tutuluyor ama sen tablo mantığı için bunu "Active" / "Passive" stringine çevirmek istiyorsun (Filtreleme kolay olsun diye).

```typescript
// Veri: { status: 1 } (1=Aktif, 0=Pasif)

{
  id: "statusText",
  header: "Durum",
  accessorFn: (row) => (row.status === 1 ? "Active" : "Inactive"),
  filterFn: "equalsString" // Filtreleme artık "Active" yazısına göre çalışır
}
```

### Çok Kritik Bir Ayrım: accessorFn vs cell

Yeni başlayanların en çok karıştırdığı nokta budur. "Veriyi cell içinde de birleştirebilirim, neden accessorFn kullanayım?"

- **accessorFn:** Tablonun BEYNİ içindir. (Sorting, Filtering, Grouping buradaki değere göre yapılır).
- **cell:** Tablonun YÜZÜ içindir. (Sadece kullanıcıya görünür, sıralamaya etki etmez).

**Hatalı Kullanım (Sıralama Bozulur):**

```typescript
{
  accessorKey: "price", // Değer: 100 (Number)
  header: "Fiyat",
  // Cell içinde "100 TL" yaparsan, sıralama hala 100 sayısına göre düzgün çalışır.
  cell: ({ row }) => `${row.getValue("price")} TL`
}
```

**Hatalı Kullanım (accessorFn ile):**

```typescript
{
  id: "price",
  header: "Fiyat",
  // DİKKAT: Sayıyı String'e çevirdin! "100 TL".
  // String sıralamasında "1000 TL", "20 TL"den önce gelir! (1 < 2 olduğu için)
  accessorFn: (row) => `${row.price} TL`,
}
```

### Özet: Nasıl Karar Veririm?

- Veritabanındaki alanın aynısını istiyorsan -> accessorKey kullan.
- Verileri birleştirip (Ad+Soyad) buna göre sıralama/filtreleme yapacaksan -> accessorFn kullan.
- Sadece görsel makyaj (TL ekleme, renkli badge yapma, tarih formatlama) yapacaksan -> cell kullan.



## header, cell, footer 
`ColumnDef` içinde her sütun için 3 farklı render noktası tanımlayabilirsin:

- header: Sütun başlığı (thead içinde). Genelde *etiket*, *sıralama butonu*, *select-all checkbox* vb.
- cell: Satır hücresi (tbody içinde). Veriyi gösterme/formatlama, satır bazlı aksiyonlar vb.
- footer: Sütun altı (tfoot içinde). Toplam, ortalama, özet bilgiler vb. (UI’da tfoot render etmiyorsan görünmez.)

---

## header kullanımı

### 1) Basit başlık (string)
```ts
{
  accessorKey: "email",
  header: "Email",
}
```

### 2) Fonksiyon ile başlık
Örneğin satır seçme checkbox’ı koymak istersen:
```js
  {
    id: 'select',
    header: ({ table }) => ( // TanStack Table'in verdiği 'table' objesini kullanabilirsin
      <Checkbox
        checked={
          table.getIsAllPageRowsSelected() ||
          (table.getIsSomePageRowsSelected() && 'indeterminate')
        }
        onCheckedChange={(value) => table.toggleAllPageRowsSelected(!!value)}
        aria-label='Select all'
        className='translate-y-[2px]'
      />
    ),
    ...
  }
```
veya sıralama ile gizle/göster butonu eklemek istersen:

```js
  {
    accessorKey: 'title',
    header: ({ column }) => (
      <DataTableColumnHeader column={column} title='Title' /> // Dropdown için custom component. TanStack'in column objesini props olarak geçiyorsun
    ),
    ...
  },
```

```js
type DataTableColumnHeaderProps<TData, TValue> =
  React.HTMLAttributes<HTMLDivElement> & {
    column: Column<TData, TValue>
    title: string
  }

export function DataTableColumnHeader<TData, TValue>({
  column,
  title,
  className,
}: DataTableColumnHeaderProps<TData, TValue>) {
  if (!column.getCanSort()) {
    return <div className={cn(className)}>{title}</div>
  }

  return (
    <div className={cn('flex items-center space-x-2', className)}>
      <DropdownMenu>
        <DropdownMenuTrigger asChild>
          <Button
            variant='ghost'
            size='sm'
            className='h-8 data-[state=open]:bg-accent'
          >
            <span>{title}</span>
            {column.getIsSorted() === 'desc' ? (
              <ArrowDownIcon className='ms-2 h-4 w-4' />
            ) : column.getIsSorted() === 'asc' ? (
              <ArrowUpIcon className='ms-2 h-4 w-4' />
            ) : (
              <CaretSortIcon className='ms-2 h-4 w-4' />
            )}
          </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent align='start'>
          <DropdownMenuItem onClick={() => column.toggleSorting(false)}>
            <ArrowUpIcon className='size-3.5 text-muted-foreground/70' />
            Asc
          </DropdownMenuItem>
          <DropdownMenuItem onClick={() => column.toggleSorting(true)}>
            <ArrowDownIcon className='size-3.5 text-muted-foreground/70' />
            Desc
          </DropdownMenuItem>
          {column.getCanHide() && (
            <>
              <DropdownMenuSeparator />
              <DropdownMenuItem onClick={() => column.toggleVisibility(false)}>
                <EyeNoneIcon className='size-3.5 text-muted-foreground/70' />
                Hide
              </DropdownMenuItem>
            </>
          )}
        </DropdownMenuContent>
      </DropdownMenu>
    </div>
  )
}
```

- `header: ({ column }) => (...)` ile TanStack’in verdiği `column` objesini alıp
- `column.toggleSorting(...)`, `column.toggleVisibility(false)`, `column.getIsSorted() === 'desc/asc'` çağırarak yönetiyorsun.
  
---

## cell kullanımı

cell her satır için ayrı ayrı çalışır. İki yaygın veri kaynağı:

- `row.getValue("kolonId")`: O kolonun “table value”sunu alır (accessorKey ile gelen değer)
- `row.original`: Satırın ham verisi (orijinal obje). Aksiyon menülerinde çok kullanılır.

### 1) Formatlama örneği (amount)
- `cell: ({row}) => row.getValue("amount")` alınıyor (amount accessorKey ile geliyor. Tanımlama düzgün yapılmalı.)
- `Intl.NumberFormat` ile para birimi formatlanıyor

### 2) Aksiyon sütunu (accessor yok, tamamen custom)
- `id: "actions"` diyorsun (bu kolonun accessor’ı yok)
- `cell: ({ row }) => ...` ile `row.original` üzerinden kullanıcıyı alıp menü açıyorsun

---

## footer kullanımı
footer, sütunun alt başlığı/özeti için kullanılır
- Toplam (`sum(amount)`)
- Ortalama
- “Seçili satır sayısı”
- Kolon adı tekrar vs.

TanStack tarafında footer tanımlaması ile birlikte tabloda görülmesi için için UI’da <TableFooter> render edilmesi gerekir. 
Örnek:
```js
{
  accessorKey: "amount",
  header: "Amount",
  footer: ({ column }) => (
    <div>
      Total: {column.getFacetedUniqueValues().size} items
    </div>
  ),
}
```

---

# TanStack Table Kolon Özellikleri (ColumnDef)

## 1) Kolonu tanımlama (ID + veri kaynağı)
- **`accessorKey`**: Satır objesinden alan okur.  
  Örn: `accessorKey: "email"`
- **`accessorFn`**: Türetilmiş/hesaplanmış değer üretir.  
  Örn: `accessorFn: (row) => row.name.toUpperCase()`
- **`id`**: Özellikle `accessorFn` varsa (veya “actions” gibi alanı olmayan kolonlarda) kullanılır.  
  Örn: `id: "actions"`

## 2) Kolon gruplama (multi-header)
- **`columns`**: Header altında alt kolonlar tanımlamak için (grouped columns).  
  Örn: “Kişi Bilgileri” → name + email

## 3) Sıralama (sorting) ile ilgili
- **`enableSorting`**: O kolonda sorting açık/kapalı.
- **`sortingFn`**: Custom sıralama fonksiyonu (metin/numeric/tarih özel durumları).
- **`sortDescFirst`**, **`invertSorting`**: Sıralama yön davranışını değiştirir.

## 4) Filtreleme ile ilgili
- **`enableColumnFilter`**: Kolon filtrelemesi açık/kapalı.
- **`filterFn`**: Custom filtre fonksiyonu (örn. `includesString`, `equals`, veya kendi fonksiyonun).
- **`enableGlobalFilter`**: Global search bu kolonu dikkate alsın mı?

> Faceted filter kullanıyorsan, kolonun filter değer tipi genelde `string[]` olur; `filterFn` ile uyumlu seçmek önemli.

## 5) Görünürlük / gizleme
- **`enableHiding`**: “View options” ile gizlenebilir mi?

## 6) Boyutlandırma (resizing)
- **`size`**, **`minSize`**, **`maxSize`**: Kolon genişliği kontrolü
- **`enableResizing`**: Bu kolon resize edilebilir mi?

## 7) Gruplama / agregasyon (özet satırlar)
(İhtiyaç olursa)
- **`enableGrouping`**
- **`aggregationFn`**
- **`aggregatedCell`**: Grup satırında hücre nasıl görünsün?

## 8) `meta` (en faydalı “ileri seviye” alan)
- **`meta`**: Kolona özel ekstra bilgi taşımak için (tamamen sana ait).  
  Örn: `meta: { align: "right", label: "Tutar" }` gibi; sonra `cell/header` içinde kullanırsın.

---

### Senin notların bağlamında “bilmen en kritik 6’lı”
`accessorKey | accessorFn | id | enableSorting | filterFn | meta`

İstersen kullandığın örneğe göre (selection, amount, actions) hangi kolonlarda hangi key’ler mantıklı olur, onu da kısa bir öneri listesi olarak yazabilirim.