# TanStack Query Global Error Handling 🚀

Proje büyüdükçe her `useQuery` veya `useMutation` yanına `onError: (err) => toast.error(err.message)` yazmak hem kod tekrarıdır (DRY prensibine aykırı) hem de bakımı kabusa çevirir.

TanStack Query, **Global Callbacks** (`QueryCache` ve `MutationCache`) ve **Meta** özelliği sayesinde bunu merkezi bir yerden yönetmemize olanak tanır.

İşte "Feature-Based" mimarimize uygun merkezi hata yönetimi kurulumu:

## 1. Global Konfigürasyonun Güncellenmesi ⚙️
Daha önce oluşturduğumuz `src/lib/react-query.ts` dosyasını güncelleyeceğiz. Burada `QueryCache` ve `MutationCache` sınıflarını kullanarak global olayları dinleyeceğiz.

Bu örnekte, hata mesajlarını göstermek için popüler bir kütüphane olan `react-hot-toast` (veya `react-toastify`) kullandığımızı varsayalım.

**src/lib/react-query.ts:**

```typescript
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query';
import toast from 'react-hot-toast'; // Kullandığınız toast kütüphanesi
import { axiosInstance } from './axios'; // Axios error tipleri için gerekebilir

// Hata mesajını ayrıştıran yardımcı fonksiyon
function getErrorMessage(error: any) {
  // Backend'den gelen standart bir hata mesajı formatınız varsa buraya göre ayarlayın
  return error.response?.data?.message || error.message || 'Bilinmeyen bir hata oluştu';
}

export const queryClient = new QueryClient({
  // 1. Sorgular (Queries) için Global Hata Yönetimi
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Eğer bu query için özel bir meta config varsa, global hatayı ezebiliriz
      if (query.meta?.errorMessage) {
        toast.error(query.meta.errorMessage as string);
      } else if (query.meta?.suppressError) {
        // Sessiz kal (Örneğin: Kullanıcı yazarken arama yapıyorsa hata gösterme)
        return; 
      } else {
        // Varsayılan davranış: Hatayı göster
        toast.error(`Veri alınamadı: ${getErrorMessage(error)}`);
      }
    },
  }),

  // 2. Değişiklikler (Mutations) için Global Hata Yönetimi
  mutationCache: new MutationCache({
    onError: (error, _variables, _context, mutation) => {
      // Mutation'lar genelde kullanıcı tetiklediği için varsayılan olarak hata göstermek iyidir
      if (mutation.meta?.suppressError) return;

      const message = mutation.meta?.errorMessage as string || getErrorMessage(error);
      toast.error(message);
    },
  }),
  
  defaultOptions: {
    queries: {
      retry: false, // Global hata yönetimi varken sonsuz döngüye girmemek için retry'ı sınırlamak iyidir
      refetchOnWindowFocus: false,
    },
  },
});
```

---

## 2. TypeScript ile Meta Tip Güvenliği 🔒
Yukarıda `query.meta?.errorMessage` kullandık. Ancak TypeScript varsayılan olarak meta objesinin boş olduğunu düşünür. Proje büyüdükçe developerların meta içine ne yazabileceğini bilmesi gerekir.

Bunun için bir TypeScript Declaration dosyası oluşturup TanStack Query tiplerini genişletmeliyiz.

**src/types/react-query.d.ts** (veya global types klasörünüze):

```typescript
import '@tanstack/react-query';

declare module '@tanstack/react-query' {
  interface Register {
    queryMeta: {
      errorMessage?: string;
      suppressError?: boolean;
    };
    mutationMeta: {
      errorMessage?: string;
      suppressError?: boolean;
    };
  }
}
```

Bu sayede IDE'niz, `meta: { ... }` yazarken size `errorMessage` ve `suppressError` seçeneklerini otomatik tamamlayacaktır.

---

## 3. Kullanım Senaryoları 📋
Artık feature klasörlerimizdeki hook'ları çok daha temiz yazabiliriz.

### Senaryo A: Varsayılan Davranış ✅
Hiçbir şey belirtmezseniz, global handler devreye girer ve backend hatasını toast olarak basar.

**src/features/products/api/use-create-product.ts:**

```typescript
export const useCreateProduct = () => {
  return useMutation({
    mutationFn: createProduct,
    // onError yazmamıza GEREK YOK. Global handler halledecek.
    onSuccess: () => {
      // Sadece başarı durumunu yönetin
      queryClient.invalidateQueries({ queryKey: productKeys.lists() });
      toast.success('Ürün oluşturuldu');
    }
  });
};
```

### Senaryo B: Özel Hata Mesajı ⚠️
Backend hatası ne olursa olsun, kullanıcıya dostane bir mesaj göstermek istiyorsunuz.

```typescript
useQuery({
  queryKey: productKeys.list(filter),
  queryFn: () => getProducts(filter),
  meta: {
    errorMessage: "Ürün listesi yüklenirken bir sorun oluştu. Lütfen sayfayı yenileyin."
  }
});
```

### Senaryo C: Global Hatayı Susturmak (Manual Handling) 🔇
Bazen hatayı global toast ile değil, formun içinde inputun altında kırmızı bir text olarak göstermek istersiniz. O zaman global handler'ı susturmalısınız.

```typescript
const { error, isError } = useMutation({
  mutationFn: loginUser,
  meta: {
    suppressError: true // Global toast çıkmasın, ben kendim yöneteceğim
  }
});

// UI içinde
return (
  <form>
     ...
     {isError && <div className="text-red-500">{error.message}</div>}
  </form>
)
```

---

## Özet: Bu Yapı Size Ne Kazandırır? 🎉
- **Temiz Kod:** Component ve Hook'larınızdaki try-catch veya onError blokları %80 oranında azalır.
- **Tutarlılık:** Uygulamanın bir yerinde hata sağ üstte çıkarken, diğer yerinde altta çıkmaz. Her yerde standart bir deneyim olur.
- **Esneklik:** İstediğiniz zaman meta etiketi ile global kuralı ezip o duruma özel davranış sergileyebilirsiniz.