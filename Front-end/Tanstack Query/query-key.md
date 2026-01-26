# TanStack Query Query Key: Kalbin Atışı 🔑

TanStack Query (React Query) kullanırken Query Key (Sorgu Anahtarı) mekanizması, kütüphanenin kalbidir. Bu anahtarlar, verinin önbellekte (cache) nasıl saklanacağını, nasıl bulunacağını, ne zaman güncelleneceğini ve hangi durumlarda "bayat" (stale) kabul edileceğini belirler.

Yanlış yapılandırılmış anahtarlar; verilerin gereksiz yere tekrar çekilmesine (over-fetching), arayüzde eski verinin kalmasına veya cache'in düzgün temizlenmemesine neden olur.

İşte TanStack Query key mekanizmasının çalışma mantığı, hiyerarşik yapısı ve gerçek dünya senaryoları için best practice yöntemleri:

## 1. Temel Mantık: Query Key Nedir? 🧠
Query Key, TanStack Query v4 ve v5 itibarıyla mutlaka bir dizi (array) olmalıdır. Bu dizi, verinizin "adresidir".

Query Key'ler deterministiktir. Yani dizinin içindeki elemanların sırası ve içeriği aynıysa, o key aynı kabul edilir.

```javascript
// ✅ Bunlar aynı key kabul edilir (Obje key sıralaması önemsizdir)
['todos', { status: 'done', page: 1 }]
['todos', { page: 1, status: 'done' }]

// ❌ Bunlar FARKLI key kabul edilir (Dizi eleman sırası önemlidir)
['todos', 'list', 1]
['todos', 1, 'list']
```

## 2. Best Practice: Hiyerarşik Yapı Kurmak 🏗️
En yaygın ve önerilen yaklaşım, key'leri "Genelden Özele" doğru hiyerarşik bir yapıda kurmaktır. Bu, özellikle veriyi geçersiz kılarken (invalidation) büyük kolaylık sağlar.

Standart bir yapı şu şekildedir: `[ 'scope', 'entity', 'action', 'id/params' ]`

**Örnek Yapı:**
- **Scope (Kapsam):** Uygulamanın hangi modülü? (örn: users, posts, todos)
- **Entity/Context (Bağlam):** Ne tür bir veri? (örn: list, detail)
- **Params (Parametreler):** Filtreler, ID'ler vb.

---

## 3. Gerçek Dünya Senaryoları ve Key Tanımları 📋
Bir E-ticaret uygulamasındaki "Ürünler" (Products) modülü üzerinden gidelim.

### Senaryo A: Tüm Ürünleri Listeleme
En genel key'dir. Genellikle sadece entity adını içerir.

```javascript
useQuery({
  queryKey: ['products'], 
  queryFn: fetchAllProducts
})
```

### Senaryo B: Filtrelenmiş Liste (Pagination & Filter)
Kullanıcı bir kategori seçtiğinde veya sayfa değiştirdiğinde, React Query bunu yeni bir veri seti olarak algılamalıdır. Bu yüzden değişkenler (dependencies) key'in içine eklenmelidir.

```javascript
const filter = { category: 'electronics', page: 2 };

useQuery({
  // Filter objesi değiştiğinde sorgu otomatik olarak tekrar çalışır.
  queryKey: ['products', 'list', filter], 
  queryFn: () => fetchProducts(filter)
})
```

Neden `list` ekledik? Çünkü ileride tekil detay verisiyle karışmasını istemiyoruz.

### Senaryo C: Tekil Ürün Detayı (Detail View)
Bir ürüne tıklandığında sadece o ürünün ID'sine özgü bir cache oluşmalıdır.

```javascript
const productId = 105;

useQuery({
  queryKey: ['products', 'detail', productId],
  queryFn: () => fetchProductDetail(productId)
})
```

---

## 4. Profesyonel Yaklaşım: Query Key Factory 🏭
Büyük projelerde key'leri manuel olarak `['products', 'list', filters]` şeklinde yazmak hataya açıktır. Bir yerde `list` yazıp diğer dosyada `lists` yazarsanız cache mekanizması bozulur.

Bunun yerine Query Key Factory pattern'i kullanılır. Bu, tip güvenliği ve tek merkezden yönetim sağlar.

**keys/product-keys.ts:**

```typescript
// keys/product-keys.ts

export const productKeys = {
  // En genel key (Root)
  all: ['products'] as const,

  // Listeler için (Root'tan türer)
  lists: () => [...productKeys.all, 'list'] as const,
  
  // Filtreli listeler (Lists'ten türer)
  list: (filters: string) => [...productKeys.lists(), { filters }] as const,

  // Detaylar için (Root'tan türer)
  details: () => [...productKeys.all, 'detail'] as const,
  
  // Spesifik detay (Details'ten türer)
  detail: (id: number) => [...productKeys.details(), id] as const,
};
```

**Kullanımı:**

```javascript
// Component içinde kullanımı çok temizdir:
useQuery({
  queryKey: productKeys.list({ page: 1 }),
  queryFn: ...
})

useQuery({
  queryKey: productKeys.detail(105),
  queryFn: ...
})
```

---

## 5. Invalidation (Geçersiz Kılma) Stratejisi 🔄
Hiyerarşik yapının gücü `invalidateQueries` fonksiyonunda ortaya çıkar. React Query "Fuzzy Matching" (Bulanık Eşleşme) kullanır.

Diyelim ki yeni bir ürün eklediniz (mutation) ve listelerin güncellenmesini istiyorsunuz.

```javascript
const queryClient = useQueryClient();

// 1. Sadece 'products' içeren HER ŞEYİ geçersiz kılar (Listeler, detaylar, her şey)
// Genellikle çok agresiftir, ama garantidir.
queryClient.invalidateQueries({ queryKey: ['products'] });

// 2. Sadece LİSTELERİ geçersiz kılar (Detay cache'leri kalır)
// queryKey: ['products', 'list', ...] ile başlayan her şeyi yeniler.
queryClient.invalidateQueries({ queryKey: ['products', 'list'] });

// 3. Sadece spesifik bir ID'yi geçersiz kılar (Örn: Ürün düzenlemesi sonrası)
queryClient.invalidateQueries({ queryKey: ['products', 'detail', 105] });
```

---

## 6. Özet Tablo: Hangi Durumda Hangi Key? 📊
| Senaryo          | Örnek Key Yapısı                  | Açıklama |
|------------------|-----------------------------------|----------|
| Genel Kaynak     | `['todos']`                       | O kaynağa ait her şeyin kökü. |
| Listeler         | `['todos', 'list']`               | Tüm liste türlerini ayırmak için. |
| Parametreli Liste| `['todos', 'list', { sort: 'date' }]` | Filtre değişince refetch tetikler. |
| Tekil Kayıt      | `['todos', 'detail', 5]`          | ID 5 olan kaydı diğerlerinden ayırır. |
| Bağımlı Sorgular | `['comments', { todoId: 5 }]`     | Bir todo'ya ait yorumlar. |

Sayfa numarası değiştiğinde, React Query mevcut key dizisini güncellemez; bunun yerine tamamen yeni ve benzersiz bir key oluşturur.

Yani bellekte (RAM/Cache) tek bir anahtarın içeriği değişmez, yanına ikinci bir anahtar açılır.

İşte bellekteki (ve React Query DevTools'daki) görüntü tam olarak şöyle olacaktır:

## Bellek (Cache) Durumu 🧠
Kullanıcı önce Sayfa 1'e, ardından Sayfa 2'ye tıkladığında Cache Map'inde (Hafızada) şu iki kayıt aynı anda durur:

```javascript
// 🟢 ÖNBELLEK (CACHE) GÖRÜNÜMÜ

[
  // 1. Kayıt (Artık "Inactive" durumda olabilir ama silinmemiştir)
  {
    queryKey: ['products', 'list', { page: 1, category: 'shoes' }],
    state: { status: 'success', data: { ...Sayfa 1 verileri... } }
  },

  // 2. Kayıt (Şu an "Active" olan, ekranda görünen)
  {
    queryKey: ['products', 'list', { page: 2, category: 'shoes' }],
    state: { status: 'success', data: { ...Sayfa 2 verileri... } }
  }
]
```

### Neden Böyle Görünüyor?
React Query, `queryKey` dizisini hash'ler (benzersiz bir kimliğe dönüştürür).

- `{ page: 1 }` içeren dizi → ID_A
- `{ page: 2 }` içeren dizi → ID_B

Bu iki ID birbirinden farklı olduğu için React Query bunları farklı veriler olarak kabul eder.

### Bu Durumun Avantajı Nedir? ✅
Kullanıcı Sayfa 2'deyken tekrar Sayfa 1 butonuna basarsa;

React Query hafızaya bakar: `['products', 'list', { page: 1 ... }]` var mı?

Evet var! (Daha önce çekmiştik).

- Loading spinner göstermez, anında (milisaniyeler içinde) eski veriyi ekrana basar.
- Arka planda (background) verinin güncel olup olmadığını kontrol eder (refetch), yeniyse günceller.

---

## UI Deneyimi İçin Kritik İpucu: placeholderData 🎨
Sayfa 1'den Sayfa 2'ye geçerken, Key değiştiği için React Query bunu "Yeni bir istek" olarak görür ve Sayfa 2 verisi gelene kadar ekranda Loading (Yükleniyor) durumu oluşturur. Bu da ekranın "yanıp sönmesine" (flicker) neden olur.

Bunu engellemek ve sanki tek bir liste akıyormuş hissi vermek için v5 ile gelen `keepPreviousData` mantığını kullanmalısınız:

```javascript
import { keepPreviousData } from '@tanstack/react-query';

const [page, setPage] = useState(1);

useQuery({
  // Key değiştikçe yeni cache girişi oluşur
  queryKey: ['products', 'list', { page }],
  queryFn: () => fetchProducts(page),
  
  // ✨ SİHİRLİ SATIR:
  // Yeni key (Sayfa 2) yüklenene kadar, eski key'in (Sayfa 1) verisini ekranda tut.
  placeholderData: keepPreviousData
});
```

---

## Sayfalama (Pagination) Performans Optimizasyonu ⚡
Sayfalama (Pagination) senaryolarında performansın kilidi tam olarak bu iki ayarın (`staleTime` ve `gcTime`) doğru yapılandırılmasında yatar.

Varsayılan ayarlarda React Query "agresif" davranır: Veriyi anında bayat (stale) kabul eder. Bu, kullanıcı Sayfa 1 -> Sayfa 2 -> Sayfa 1 yaptığında, veriyi ekranda anında gösterse bile arka planda hemen sunucuya bir istek daha atıp "Değişen bir şey var mı?" diye kontrol edeceği anlamına gelir.

Bunu optimize etmek için şu iki kavramı netleştirelim:

### 1. staleTime (Bayatlama Süresi) ⏱️
"Veri ne kadar süre taze kalsın ve sunucuya sorulmasın?"

- **Varsayılan:** 0 ms (Veri geldiği an bayattır).
- **Sayfalama İçin Öneri:** Genellikle 1 dakika ile 5 dakika arası.

**Senaryo:**
Kullanıcı bir e-ticaret sitesinde ürünleri geziyor. Sayfa 1'e baktı, Sayfa 2'ye geçti. 30 saniye sonra tekrar Sayfa 1'e döndü.

- **staleTime: 0 (Varsayılan):** React Query, Sayfa 1'i cache'ten gösterir AMA arka planda tekrar fetch atar. Network tabında gereksiz bir trafik oluşur.
- **staleTime: 60000 (1 dk):** React Query bakar, "Bu veriyi çekeli daha 30 saniye oldu, süresi dolmadı" der. Hiçbir network isteği atmaz. Tamamen cache'ten çalışır.

### 2. gcTime (Eski adıyla cacheTime) 🗑️
"Kullanıcı bu veriyi kullanmayı bıraktıktan sonra (Inactive), hafızada ne kadar tutayım?"

- **Varsayılan:** 5 dakika.
- **Sayfalama İçin Öneri:** Varsayılan (5 dk) genelde iyidir ama kullanıcı çok uzun süre geziniyorsa 10-15 dakikaya çıkarılabilir.

**Senaryo:**
Kullanıcı "Ürünler" sayfasından tamamen çıkıp "Profil" sayfasına gitti.

React Query bir sayaç başlatır. Eğer kullanıcı 5 dakika içinde tekrar ürünlere dönmezse, o sayfalara ait (Page 1, Page 2 vs.) tüm verileri RAM'den siler (Garbage Collection).

---

## Kod Üzerinde Uygulama 💻
Sayfalama yaparken genellikle verinin anlık değişmesi (borsa verisi değilse) kritik değildir. Kullanıcıya hızlı bir deneyim sunmak için `staleTime` mutlaka tanımlanmalıdır.

```javascript
useQuery({
  queryKey: ['products', 'list', { page }],
  queryFn: () => fetchProducts(page),
  
  // ✅ 1 dakika boyunca bu sayfaya geri dönülürse sunucuya gitme.
  staleTime: 1000 * 60 * 1, 
  
  // ✅ Kullanıcı bu sayfadan ayrılırsa veriyi 10 dakika boyunca hafızada tut (silme).
  gcTime: 1000 * 60 * 10, 
  
  // ✅ Sayfa geçişlerinde flicker'ı önlemek için (önceki konuşmamızdan).
  placeholderData: keepPreviousData, 
});
```

---

## Özet Karşılaştırma Tablosu 📊
| Durum | staleTime Dolmamış (Fresh) | staleTime Dolmuş (Stale) |
|-------|-----------------------------|---------------------------|
| Kullanıcı Sayfaya Geri Döndü | 🟢 Veri Cache'ten gelir. 🛑 Network isteği YOK. | 🟡 Veri Cache'ten gelir. 🔄 Arka planda Network isteği atılır. |
| Loading Spinner? | Görünmez (Anlık). | Görünmez (Anlık). |
| Veri Güncelliği | Eski olabilir (kabul edilebilir risk). | Güncellenir. |

---

## Pro Tip: prefetchQuery ile Geleceği Tahmin Etmek 🔮
Sayfalama performansını "mükemmel" seviyeye çıkarmak istiyorsanız, kullanıcı Sayfa 1'deyken arka planda gizlice Sayfa 2'yi yükleyebilirsiniz.

Böylece kullanıcı "İleri" butonuna bastığında veri çoktan inmiş olur (`staleTime` ayarlı olduğu için tekrar çekmez) ve geçiş 0 ms sürer.

```javascript
// Component içinde
const queryClient = useQueryClient();

useEffect(() => {
  // Kullanıcı şu anki sayfayı (page) görüntülerken,
  // bir sonraki sayfanın (page + 1) verisini şimdiden çekip cache'e atıyoruz.
  if (hasMore) {
    queryClient.prefetchQuery({
      queryKey: ['products', 'list', { page: page + 1 }],
      queryFn: () => fetchProducts(page + 1),
      staleTime: 1000 * 60 * 1 // Prefetch edilen veri de 1 dk taze kalsın
    });
  }
}, [page, queryClient]);
```