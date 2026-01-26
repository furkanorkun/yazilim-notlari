# TanStack Query Select: Gizli Süper Güç 🚀

`select` seçeneği, TanStack Query'nin "gizli süper gücü" olarak bilinir. Genellikle gözden kaçar ama mimari temizliği ve performans için hayati öneme sahiptir.

Basitçe anlatmak gerekirse: `select`, sunucudan gelen ham veriyi (Raw Data), component'in ihtiyaç duyduğu formata dönüştüren bir "filtre" veya "dönüştürücü" katmandır.

## Mantığı Şöyle Çalışır 🔄
Normalde akış şöyledir: **Backend (Veri) -> TanStack Query (Cache) -> Component**

`select` kullandığında akış şöyle olur: **Backend (Veri) -> TanStack Query (Cache) -> SELECT (Dönüştürme) -> Component**

Yani veri cache'te ham haliyle durur (böylece başka componentler ham halini kullanabilir), ama senin component'ine gelmeden önce istediğin şekle girer.

## Neden Kullanmalısın? (Avantajları) ✅
- **Performans (Memoization):** Eğer backend'den devasa bir JSON geliyorsa ve sen sadece içinden 2 alanı alıp kullanacaksan, component'in her render'ında bu büyük objeyi işlemek yerine, `select` bunu sadece veri değiştiğinde yapar.
- **Backend Bağımlılığını Azaltma:** Backend'den gelen veri yapısı kötü olabilir (örneğin tarihler string geliyordur, iç içe objeler vardır). `select` ile bu veriyi temizleyip UI'a tertemiz (clean) bir veri sunarsın.
- **Tek İstek, Çok Görünüm:** API'den "Tüm Ürünleri" bir kere çekersiniz. Bir component sadece "Aktif Ürünleri" listeler, diğeri sadece "Ürün İsimlerini". İkisi için ayrı API isteği atmazsınız.

## Örnek Senaryo 📋
Diyelim ki backend'den şöyle bir ürün listesi geliyor:

```json
[
  { "id": 1, "name": "Laptop", "price": 1000, "is_active": true, "category_id": 5 },
  { "id": 2, "name": "Mouse", "price": 50, "is_active": false, "category_id": 5 },
  { "id": 3, "name": "Klavye", "price": 100, "is_active": true, "category_id": 5 }
]
```

Ama senin component'in (örneğin bir Dropdown menüsü) sadece şuna ihtiyaç duyuyor: `[{ label: "Laptop", value: 1 }, { label: "Klavye", value: 3 }]`

Bunu component içinde `.map` ve `.filter` ile yaparsan her render'da tekrar hesaplanır. `select` ile bunu kaynağında hallederiz.

---

## Mimariye Entegre Edelim (Best Practice) 🏗️
Daha önce yazdığımız `useProducts` hook'unu, `select` opsiyonunu destekleyecek şekilde güncelleyelim. Bu biraz TypeScript "Generics" büyüsü gerektirir ama çok güçlüdür.

**src/features/products/api/use-products.ts:**

```typescript
import { useQuery } from '@tanstack/react-query';
import { getProducts, GetProductsParams } from './get-products';
import { productKeys } from './product-keys';
import { Product } from '../types';

type UseProductsOptions<TData = Product[]> = {
  params: GetProductsParams;
  enabled?: boolean;
  // Burada diyoruz ki: select fonksiyonu Product[] alır, dışarı TData (her ne isterse) döner.
  select?: (data: Product[]) => TData; 
};

// Hook'u generic hale getirdik <TData = Product[]>
export const useProducts = <TData = Product[]>({ 
  params, 
  enabled = true, 
  select 
}: UseProductsOptions<TData>) => {
  
  return useQuery({
    queryKey: productKeys.list(JSON.stringify(params)),
    queryFn: () => getProducts(params),
    enabled,
    staleTime: 1000 * 60 * 5,
    // İşte sihirli kısım burada:
    select: select, 
  });
};
```

## Component İçinde Kullanımı 🎯
Artık aynı hook'u farklı şekillerde kullanabiliriz.

### 1. Normal Kullanım (Tüm Datayı Alır)
Hiçbir şey değiştirmezseniz `Product[]` döner.

```typescript
const { data } = useProducts({ params: { page: 1 } });
// data tipi: Product[]
```

### 2. Select Kullanımı (Dönüştürülmüş Veri)
Dropdown için veriyi filtreleyip şeklini değiştiriyoruz.

```typescript
export const ProductSelectBox = () => {
  const { data } = useProducts({ 
    params: { page: 1 },
    // Select fonksiyonu: Sadece aktifleri al ve şeklini değiştir
    select: (products) => 
      products
        .filter(p => p.is_active)
        .map(p => ({
          label: p.name,
          value: p.id
        }))
  });

  // TypeScript buradaki data'nın tipinin otomatik olarak 
  // { label: string, value: number }[] olduğunu anlar!
  
  return (
    <select>
      {data?.map(opt => (
        <option key={opt.value} value={opt.value}>{opt.label}</option>
      ))}
    </select>
  );
};
```

### 3. Select Kullanımı (Sadece Tek Bir Değer Hesaplama)
Örneğin toplam stok değerini hesaplamak istiyorsunuz.

```typescript
const { data: totalPrice } = useProducts({
  params: {},
  // Tüm array'i tek bir number'a indirge
  select: (products) => products.reduce((acc, curr) => acc + curr.price, 0)
});
// totalPrice tipi: number
```

## Önemli Bir Püf Noktası: useCallback ⚠️
Eğer `select` içine yazdığınız fonksiyon çok karmaşıksa veya component içinde satır içi (inline) olarak tanımladıysanız (`select: (data) => ...`), React her render'da yeni bir fonksiyon oluşturur. Bu da `select`'in optimizasyonunu bozabilir.

Eğer dönüşüm işlemi ağırsa, fonksiyonu ya component dışında tanımlayın ya da `useCallback` ile sarmalayın.

Basit dönüşümler için (yukarıdaki `map`/`filter` gibi) buna gerek yoktur, TanStack Query yeterince akıllıdır. Ancak çok ağır matematiksel işlemler yapacaksanız:

```typescript
// Component dışına taşımak en temizi (Stable Reference)
const selectActiveProductLabels = (data: Product[]) => 
  data.filter(d => d.isActive).map(d => d.name);

const MyComponent = () => {
  const { data } = useProducts({ 
    params: {}, 
    select: selectActiveProductLabels // Referans hep aynı, performans süper
  });
  ...
}
```

---

## Özetle 📝
- **select Nedir?:** Veriyi mutfaktan (API) masaya (Component) getirirken yapılan son hazırlık/süsleme aşamasıdır.
- **Ne Zaman Kullanılır?:** API verisi component'in istediği formatta değilse veya büyük verinin sadece bir kısmına ihtiyacınız varsa.
- **Mimari:** Custom Hook'unuzu Generic yaparak, componentlerin veriyi istediği gibi çekip bükmesine izin verirsiniz.

Bu özellik, uygulamanızı "Veriyi olduğu gibi kullanan" bir yapıdan "Veriyi ihtiyaca göre işleyen" profesyonel bir yapıya taşır.