# TanStack Query Pre-Fetching: Hızlıdan Anlığa 🚀

Pre-fetching (Ön Yükleme), bir uygulamanın "hızlı" hissettirmesi ile "anlık" hissettirmesi arasındaki farkı yaratan tekniktir.

Mantığı çok basittir: Kullanıcı veriye ihtiyaç duymadan (tıklamadan) önce, onun ne yapacağını tahmin edip veriyi arka planda yüklemektir.

Böylece kullanıcı tıkladığında, veri zaten cache'te (hafızada) hazır beklediği için yükleme ekranı (spinner) görmez. Sayfa geçişi 0 saniye sürer.

## Nasıl Çalışır? 🔄
Normal `useQuery` bir component mount olduğunda çalışır. `prefetchQuery` ise bir component'e bağlı değildir, doğrudan `queryClient` üzerinden tetiklenir ve veriyi alıp cache'e koyar.

Bunu mevcut Feature-Based mimarimize entegre etmenin iki yaygın yolu vardır:

## 1. Senaryo: Hover ile Ön Yükleme (En Yaygın Yöntem) 🖱️
Kullanıcı faresini bir ürün kartının üzerine getirdiğinde (Hover), %80 ihtimalle o ürüne tıklayacaktır. Biz o 200-300 milisaniyelik arada veriyi çekeriz.

**src/features/products/components/product-card.tsx:**

```typescript
import { useQueryClient } from '@tanstack/react-query';
import { Link } from 'react-router-dom';
import { productKeys } from '../api/product-keys'; // Key Factory yine başrolde
import { getProductDetail } from '../api/get-product-detail'; // Fetcher fonksiyonu

type ProductCardProps = {
  id: number;
  name: string;
};

export const ProductCard = ({ id, name }: ProductCardProps) => {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    // PRE-FETCH İŞLEMİ BURADA
    queryClient.prefetchQuery({
      queryKey: productKeys.detail(id), // Standart key kullanımı
      queryFn: () => getProductDetail(id), // Standart fetcher kullanımı
      staleTime: 1000 * 60, // 1 dakika boyunca bu veri taze sayılsın (tekrar çekme)
    });
  };

  return (
    <div 
      className="card" 
      onMouseEnter={handleMouseEnter} // Mouse üzerine gelince tetikle
    >
      <h3>{name}</h3>
      <Link to={`/products/${id}`}>Detay Gör</Link>
    </div>
  );
};
```

**Sonuç:** Kullanıcı "Detay Gör" butonuna tıklayıp detay sayfasına gittiğinde, detay sayfası `useProductDetail(id)` hook'unu çalıştırır. Hook cache'e bakar, veriyi orada hazır bulur ve network isteği atmadan anında ekrana basar.

---

## 2. Senaryo: Sayfalama (Pagination) Ön Yüklemesi 📄
Kullanıcı şu an "Sayfa 1"de ise, bir sonraki adımda %90 ihtimalle "Sayfa 2"ye geçecektir. Kullanıcı Sayfa 1'i okurken biz arkada Sayfa 2'yi hazırlarız.

Bunu `use-products.ts` hook'umuzun içine veya listeleme component'ine entegre edebiliriz.

**src/features/products/components/product-list.tsx:**

```typescript
import { useState } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { useProducts } from '../api/use-products';
import { productKeys } from '../api/product-keys';
import { getProducts } from '../api/get-products';

export const ProductList = () => {
  const [page, setPage] = useState(1);
  const queryClient = useQueryClient();

  // Şu anki sayfanın verisi
  const { data, isPending } = useProducts({ params: { page } });

  // SONRAKİ SAYFAYI PRE-FETCH ETME
  // Component render olduğunda çalışır
  if (!isPending) {
    const nextPage = page + 1;
    queryClient.prefetchQuery({
      queryKey: productKeys.list(JSON.stringify({ page: nextPage })),
      queryFn: () => getProducts({ page: nextPage }),
    });
  }

  if (isPending) return <div>Yükleniyor...</div>;

  return (
    <div>
      {/* Ürün Listesi UI */}
      {data?.map(product => <div key={product.id}>{product.name}</div>)}

      <button onClick={() => setPage(old => old + 1)}>
        Sonraki Sayfa
      </button>
    </div>
  );
};
```

**Sonuç:** Kullanıcı "Sonraki Sayfa"ya bastığında bekleme süresi olmaz.

---

## Kritik Nokta: staleTime Ayarı ⚠️
Pre-fetching yaparken `staleTime` (tazelik süresi) hayati önem taşır.

- **Hata:** Eğer `staleTime` vermezseniz (varsayılan 0), veriyi prefetch edersiniz, cache'e girer ve o an "bayat" damgası yer. Kullanıcı sayfaya girdiğinde component "Veri var ama bayat, ben yine de arka planda bir daha çekeyim" der. Görüntü hızlı gelir ama gereksiz yere sunucuya istek atılır.
- **Doğrusu:** `staleTime` vererek (örneğin 10 saniye veya 1 dakika), "Bu veriyi az önce çektim, kullanıcı sayfaya girerse tekrar sunucuya sorma, bunu kullan" demiş olursunuz.