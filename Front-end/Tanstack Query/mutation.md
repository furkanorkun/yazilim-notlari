# 🚀 Tanstack Query ile Mutation (Veri Değiştirme)

Tanstack Query ile **mutation** (veri değiştirme: ekleme, güncelleme, silme) işlemlerini yönetmek, uygulamanın kullanıcı deneyimini (UX) belirleyen en kritik kısımdır.

Tıpkı veri çekerken olduğu gibi, mutation işlemlerini de feature klasörleri altında, **Custom Hook** olarak izole edeceğiz.

İki ana stratejimiz var:

- **Invalidation (Standart Yöntem)**: İşlem başarılı olursa veriyi sunucudan tekrar çek.
- **Optimistic Updates (İyimser Güncelleme)**: Sunucu yanıtını beklemeden UI'ı güncelle.

Önceki products örneği üzerinden devam edelim.

---

## 📁 1. Dosyalama Yapısı

Mutation hook'larını da api klasöründe tutuyoruz. İsimlendirmede eylemi açıkça belirtmek (create, update, delete) best practice'tir.

```
src/features/products/
├── api/
│   ├── ... (get-products vb.)
│   ├── create-product.ts      # API isteği (Axios)
│   └── use-create-product.ts  # Mutation Hook Logic
```

---

## 🔗 2. API İstek Katmanı (Fetcher)

Önce saf API fonksiyonunu yazalım.

**src/features/products/api/create-product.ts:**

```typescript
import { axiosInstance } from '@/lib/axios';
import { Product } from '../types';

export type CreateProductDTO = {
  name: string;
  price: number;
  description: string;
};

export const createProduct = async (data: CreateProductDTO): Promise<Product> => {
  const response = await axiosInstance.post('/products', data);
  return response.data;
};
```

---

## ✅ 3. Yöntem A: Invalidation (Güvenli ve Yaygın Yöntem)

Bu yöntem, *"Veriyi kaydettim, şimdi listeyi tazeleyelim"* mantığıdır. Veri tutarlılığı %100'dür ancak kullanıcı listeyi görmek için tekrar yükleme (loading) süresini bekler.

**src/features/products/api/use-create-product.ts:**

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { createProduct, CreateProductDTO } from './create-product';
import { productKeys } from './product-keys';

export const useCreateProduct = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newProduct: CreateProductDTO) => createProduct(newProduct),
    
    // İşlem başarılı olduğunda çalışır
    onSuccess: () => {
      // 1. Önceki "products" listesi bayat (stale) olarak işaretlenir.
      // 2. Eğer o sırada ekranda bir ürün listesi varsa, otomatik olarak refetch tetiklenir.
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
      
      // İsterseniz burada bir Toast/Bildirim gösterebilirsiniz.
      // toast.success('Ürün başarıyla eklendi!');
    },
  });
};
```

---

## ⚡ 4. Yöntem B: Optimistic Updates (Üst Düzey UX)

Bu yöntem, *"Kullanıcıyı bekletme, sanki işlem başarılı olmuş gibi listeye ekle, arka planda sunucuyla konuş"* mantığıdır. WhatsApp mesajlarının "tek tık" olup sonra "çift tık" olması gibidir. Biraz daha karmaşıktır ama çok daha profesyonel hissettirir.

Aynı dosyayı (use-create-product.ts) Optimistic Update için şöyle revize ederiz:

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { createProduct, CreateProductDTO } from './create-product';
import { productKeys } from './product-keys';
import { Product } from '../types';

export const useCreateProduct = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createProduct,

    // Mutation BAŞLAMADAN HEMEN ÖNCE çalışır
    onMutate: async (newProduct) => {
      // 1. Olası çakışmaları önlemek için ilgili query'leri iptal et
      await queryClient.cancelQueries({ queryKey: productKeys.lists() });

      // 2. Hata durumunda geri dönmek (rollback) için mevcut veriyi sakla
      const previousProducts = queryClient.getQueryData<Product[]>(productKeys.list(''));

      // 3. Cache'i manuel olarak güncelle (UI anında değişir)
      if (previousProducts) {
        queryClient.setQueryData(productKeys.list(''), (old: Product[] = []) => [
          ...old,
          { 
            id: Math.random(), // Geçici bir ID (veya temp-id)
            ...newProduct,
            createdAt: new Date().toISOString() 
          },
        ]);
      }

      // Context objesi döndürerek onError'da kullanabiliriz
      return { previousProducts };
    },

    // Hata olursa çalışır
    onError: (err, newProduct, context) => {
      // Cache'i eski haline döndür (Rollback)
      if (context?.previousProducts) {
        queryClient.setQueryData(productKeys.list(''), context.previousProducts);
      }
    },

    // Başarılı da olsa, hatalı da olsa işlem bitince çalışır
    onSettled: () => {
      // Veri tutarlılığını garantiye almak için sunucudan en güncel halini çek
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
    },
  });
};
```

---

## 🛠️ 5. Component İçinde Kullanım

Hangi yöntemi seçerseniz seçin (Invalidation veya Optimistic), component içindeki kullanımınız değişmez. Logic tamamen hook içinde gizlenmiştir.

```typescript
import { useCreateProduct } from '@/features/products/api/use-create-product';

export const CreateProductForm = () => {
  // Hook'u çağır
  const createProductMutation = useCreateProduct();

  const handleSubmit = (formData) => {
    createProductMutation.mutate({
      name: formData.name,
      price: Number(formData.price),
      description: formData.description
    });
  };

  return (
    <form onSubmit={...}>
      {/* Loading durumunu hook'tan alıyoruz */}
      <button disabled={createProductMutation.isPending}>
        {createProductMutation.isPending ? 'Ekleniyor...' : 'Kaydet'}
      </button>

      {/* Hata durumunu hook'tan alıyoruz */}
      {createProductMutation.isError && (
        <p>Hata: {createProductMutation.error.message}</p>
      )}
    </form>
  );
};
```

---

## 📋 Özetle Dikkat Edilmesi Gerekenler

| Konu | Açıklama |
|------|----------|
| **Query Key Factory Kullanımı** | Mutation içinde `invalidateQueries` veya `setQueryData` kullanırken mutlaka daha önce oluşturduğumuz `productKeys` objesini kullanın. Elle string yazmayın. |
| **Context Dönüşü** | Optimistic update yaparken `onMutate` fonksiyonundan eski veriyi (`previousProducts`) return etmeyi unutmayın, yoksa hata anında geri alamazsınız. |
| **onSettled** | Optimistic update yapsanız bile, işlemin sonunda `invalidateQueries` yapmak en güvenlisidir. Sunucuda ID oluşmuş olabilir veya başka bir veri değişmiş olabilir. |