Tanstack Query (eski adıyla React Query) ile büyük ölçekli ve sürdürülebilir bir proje geliştirmek istiyorsanız, "Feature-Based" (Özellik Tabanlı) dosyalama yapısı ve "Custom Hook" mimarisi en iyi yaklaşımdır.

## Dosyalama Yapısı (Folder Structure)

```plaintext
src/
├── lib/
│   ├── axios.ts           # Axios instance (interceptorlar burada)
│   └── react-query.ts     # QueryClient konfigürasyonu
├── features/
│   ├── auth/              # Auth özelliği
│   └── products/          # Ürünler özelliği
│       ├── components/    # Sadece buraya özel UI bileşenleri
│       ├── api/           # BU FEATURE'A ÖZEL QUERY'LER BURADA
│       │   ├── get-products.ts  # API fetch fonksiyonu ve tipleri
│       │   ├── use-products.ts  # Custom Hook (Tanstack Query logic)
│       │   └── product-keys.ts  # Query Key Factory (Aşağıda açıklanacak)
│       └── types/         # Ürün ile ilgili TypeScript interfaceleri
└── components/            # Global/Paylaşılan bileşenler
```

## Mimari Prensipler ve Kod Örnekleri

1. Query Key Yönetimi (Query Key Factory Pattern)

Büyük projelerdeki en büyük sorun, ['products', 'list', { filter: 'active' }] gibi array'lerin elle yazılmasıdır. Bir harf hatası cache'in çalışmamasına neden olur. Bunun yerine Query Key Factory kullanın.

src/features/products/api/product-keys.ts:

```typescript
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: string) => [...productKeys.lists(), { filters }] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: number) => [...productKeys.details(), id] as const,
};
```

Bu sayede component içinde productKeys.detail(5) diyerek standart bir key alırsınız.

2. API İstek Katmanı (Fetcher)

Tanstack Query, veriyi nasıl çektiğinizi bilmez. Bu işi saf bir fonksiyon yapmalıdır.

src/features/products/api/get-products.ts:

```typescript
import { axiosInstance } from '@/lib/axios';
import { Product } from '../types';

// Parametre ve Response tipleri
export type GetProductsParams = {
  category?: string;
  page?: number;
};

export const getProducts = async (params: GetProductsParams): Promise<Product[]> => {
  const { data } = await axiosInstance.get('/products', { params });
  return data;
};
```

3. Custom Hook Katmanı (Best Practice)
   
Componentlerinizde asla doğrudan useQuery kullanmayın. Mutlaka bir Custom Hook yazın. Bu, business logic'i UI'dan ayırır ve tekrar kullanılabilirlik sağlar.

src/features/products/api/use-products.ts:

```typescript
import { useQuery, keepPreviousData } from '@tanstack/react-query';
import { getProducts, GetProductsParams } from './get-products';
import { productKeys } from './product-keys';

// Config tipini de alarak esneklik sağlayabiliriz
type UseProductsOptions = {
  params: GetProductsParams;
  enabled?: boolean;
};

export const useProducts = ({ params, enabled = true }: UseProductsOptions) => {
  return useQuery({
    // Query Key Factory kullanımı:
    queryKey: productKeys.list(JSON.stringify(params)), 
    
    // Fetcher kullanımı:
    queryFn: () => getProducts(params),
    
    // Konfigürasyonlar:
    enabled, // İstek koşullu mu?
    staleTime: 1000 * 60 * 5, // 5 dakika boyunca veri taze kalsın (backend'e gitme)
    placeholderData: keepPreviousData, // Sayfalama yaparken UI titremesini engeller
  });
};
```

4. Global Konfigürasyon (lib/react-query.ts)

Her sorgu için tek tek retry, refetchOnWindowFocus gibi ayarları yapmak yerine, global bir default belirleyin.

src/lib/react-query.ts:

```typescript
import { QueryClient, DefaultOptions } from '@tanstack/react-query';

const queryConfig: DefaultOptions = {
  queries: {
    staleTime: 1000 * 60, // Default 1 dakika
    refetchOnWindowFocus: false, // Kullanıcı tab değiştirip dönünce tekrar çekmesin
    retry: 1, // Hata alırsan 1 kere daha dene
  },
};

export const queryClient = new QueryClient({ defaultOptions: queryConfig });
```