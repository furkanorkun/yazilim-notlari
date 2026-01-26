# Mock Service Worker ile API Simülasyonu

## Giriş

Mock Service Worker (MSW), geliştirme ve test aşamalarında API çağrılarını simüle etmek için güçlü bir araçtır. Service Worker aracılığıyla ağ seviyesinde (network level) istekleri yakalayarak, fetch veya axios fonksiyonlarını doğrudan değiştirmeden gerçek API ile iletişim kuruyor gibi davranmanızı sağlar.

Bu yaklaşımın temel avantajları:
- **Gerçekçi ortam**: Uygulamanız sanki gerçek bir sunucuyla konuşuyormuş gibi davranır
- **Network sekmeleri**: Browser'ın "Network" tabında istekleri görebilirsiniz
- **Kolay test yönetimi**: Farklı senaryo ve hata durumlarını kolayca simüle edebilirsiniz
- **Backend bağımsızlık**: Frontend geliştirmesi backend'in bitmesini beklemez

## Kurulum ve Yapılandırma

Paketi yükleyelim ve Service Worker dosyasını public klasörüne oluşturalım:

```bash
npm install msw --save-dev
npx msw init public/ --save
```

**Not:** Bu komut public klasörüne `mockServiceWorker.js` dosyasını oluşturacaktır.

## Proje Yapısı

Mock dosyalarını ana kodlarınızdan ayrı tutarak daha düzenli bir proje yapısı oluşturun. Geleneksel olarak `src/mocks` klasörü kullanılır:

```plaintext
src/
  mocks/
    ├── handlers.ts       # API endpoint tanımları
    ├── browser.ts        # Browser tarafı worker kurulumu
    └── index.ts          # Dışa aktarımları yönetme (opsiyonel)
```

## Handler Tanımları

`handlers.ts` dosyasında API cevaplarını tanımlayacağımız ana mantık yer alır. Best practice olarak, gerçek backend'in veri yapısına birebir uygun mock datalar dönmelisiniz:

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

// Tip tanımı (Best Practice: Bunu shared types dosyasından çekin)
type User = {
  id: number
  name: string
  role: 'admin' | 'user'
}

export const handlers = [
  // GET İsteği Yakalama
  http.get('https://api.example.com/users', () => {
    return HttpResponse.json<User[]>([
      { id: 1, name: 'Ahmet Yılmaz', role: 'admin' },
      { id: 2, name: 'Ayşe Demir', role: 'user' },
    ])
  }),

  // POST İsteği Yakalama - Login Örneği
  http.post('https://api.example.com/login', async ({ request }) => {
    const requestBody = await request.json() as { email: string }

    if (requestBody.email === 'admin@example.com') {
      return HttpResponse.json({
        token: 'abc-123-xyz',
        user: { name: 'Admin User' }
      })
    }

    // Hata senaryosu - 401 Unauthorized
    return new HttpResponse(null, { status: 401 })
  }),
]
```


## Worker Kurulumu

Geliştirme aşamasında (localhost'ta çalışırken) tarayıcıda çalışacak worker'ı kuruyoruz:

```typescript
// src/mocks/browser.ts
import { setupWorker } from 'msw/browser'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
```

## Uygulamaya Entegrasyon

MSW'yi sadece development ortamında çalıştırmalısınız. Production bundle'ında mock kodları bulunmamalıdır:

```typescript
// src/main.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

async function enableMocking() {
  if (import.meta.env.MODE !== 'development') {
    return
  }

  const { worker } = await import('./mocks/browser')

  return worker.start({
    onUnhandledRequest: 'bypass',
  })
}

enableMocking().then(() => {
  ReactDOM.createRoot(document.getElementById('root')!).render(
    <React.StrictMode>
      <App />
    </React.StrictMode>,
  )
})
```

## Örnek Kullanım

Uygulamada normal fetch istekleri kullanabilirsiniz. MSW bu istekleri otomatik olarak yakalayacaktır:

```ts
// src/components/UsersList.tsx
import { useEffect, useState } from 'react'

export const UsersList = () => {
  const [users, setUsers] = useState([])

  useEffect(() => {
    fetch('https://api.example.com/users')
      .then((res) => res.json())
      .then((data) => setUsers(data))
      .catch((err) => console.error('Hata:', err))
  }, [])

  return (
    <ul>
      {users.map((user: any) => (
        <li key={user.id}>{user.name} ({user.role})</li>
      ))}
    </ul>
  )
}
```

## Best Practices

Aşağıdaki uygulamaları takip ederek MSW'nin en verimli şekilde kullanabilirsiniz:

### 1. Network Hatalarını Simüle Edin

Sadece başarılı durumları (happy path) test etmek yeterli değildir. Farklı hata senaryolarını da simulate edin:

```typescript
http.get('https://api.example.com/users', ({ request }) => {
  // 500 Internal Server Error
  return new HttpResponse(null, { status: 500 })
})

http.get('https://api.example.com/users/:id', ({ params }) => {
  // 404 Not Found
  return new HttpResponse(null, { status: 404 })
})
```

### 2. Mock Data Yönetimi

Mock verileri handlers dosyasına gömünü yerine ayrı dosyalarda yönetin:

```ts
// src/mocks/data/users.json
[
  { id: 1, name: 'Ahmet Yılmaz', role: 'admin' },
  { id: 2, name: 'Ayşe Demir', role: 'user' }
]

// src/mocks/handlers.ts
import users from './data/users.json'

export const handlers = [
  http.get('https://api.example.com/users', () => {
    return HttpResponse.json(users)
  }),
]
```

Dinamik veriler için `@faker-js/faker` gibi kütüphaneleri kullanarak daha gerçekçi test verileri oluşturun.

### 3. Production Koruması

En önemli nokta: MSW'nin production'da çalışmadığından emin olun:

```ts
if (import.meta.env.MODE !== 'development') {
  return  // Development dışında hiçbir şey yapma
}
```

### 4. Network Sekmelesi

Kurulum tamamlandıktan sonra browser konsolunda şu mesajı göreceksiniz:

```
[MSW] Mocking enabled
```

Network sekmesinde isteklerinizin yanında `(ServiceWorker)` ibaresi belirecektir.

## Handler Seçim Mekanizması

MSW, hangi handler'ı çalıştıracağını belirlerken 3 temel kuralı sırasıyla kontrol eder. Bu kuralları anlamak, doğru çalışan mock API'ler yazmanın anahtarıdır.

### 1. HTTP Metodu ve Yol Eşleşmesi

MSW, sadece URL'e bakmaz; **HTTP Metodu (GET, POST, DELETE, vb.) + URL kombinasyonuna** bakar. Aynı URL için farklı metodlara sahip birden fazla handler yazabilirsiniz:

```typescript
export const handlers = [
  // Sadece GET isteklerini yakalar
  http.get('/users', () => {
    return HttpResponse.json([{ id: 1, name: 'Ahmet' }])
  }),
  
  // Sadece POST isteklerini yakalar (URL aynı olsa bile karışmaz)
  http.post('/users', () => {
    return HttpResponse.json({ success: true })
  }),

  // DELETE isteğini yakalar
  http.delete('/users/:id', () => {
    return HttpResponse.json({ deleted: true })
  }),
]
```

### 2. Sıralama Önemlidir - Order Matters

MSW, handlers dizisini **yukarıdan aşağıya doğru tarar**. Eşleşen **ilk handler'ı çalıştırır** ve durur. Bu, özellikle dinamik parametreler (path parameters) kullandığınızda kritiktir.

#### ❌ Yanlış Senaryo

```typescript
export const handlers = [
  // Bu handler '/users/admin' isteğini de yakalar!
  // Çünkü ':id' her şeyi kabul eder ("admin" kelimesini de id sanar)
  http.get('/users/:id', ({ params }) => { 
    return HttpResponse.json({ id: params.id, type: 'user' }) 
  }),

  // Bu kod asla çalışmaz (Unreachable Code) ⚠️
  http.get('/users/admin', () => { 
    return HttpResponse.json({ role: 'Super Admin' }) 
  }),
]
```

**Sonuç:** `/users/admin` isteği, id="admin" olarak yakalanır, ikinci handler asla çalışmaz.

#### ✅ Doğru Senaryo

```typescript
export const handlers = [
  // Önce kesin eşleşmelere bakılır
  http.get('/users/admin', () => { 
    return HttpResponse.json({ role: 'Super Admin' }) 
  }),

  // Sonra dinamik/genel olanlara bakılır
  http.get('/users/:id', ({ params }) => { 
    return HttpResponse.json({ id: params.id, type: 'user' }) 
  }),
]
```

**Sonuç:** `/users/admin` tam eşleşmeye gider, `/users/123` ise `:id` parametresine gider.

### 3. Query Parametreleri Eşleşmeye Dahil Değildir

Bu, MSW'de en çok kafa karıştıran yerlerden biridir. Handler tanımlarken URL'e **soru işaretiyle parametre yazılmaz**.

#### ❌ Yanlış

```typescript
// Bu yazım MSW'de çalışmaz!
http.get('/products?category=shoes')
```

#### ✅ Doğru

```typescript
http.get('/products', ({ request }) => {
  // Query parametrelerini handler'ın içinde kontrol edersiniz
  const url = new URL(request.url)
  const category = url.searchParams.get('category')

  if (category === 'shoes') {
    return HttpResponse.json(['Nike', 'Adidas'])
  }

  return HttpResponse.json(['Tüm Ürünler'])
})
```

---

## Dinamik Verilerle Çalışmak

MSW'de dinamik verileri yakalamak için iki ana yöntem vardır: **Path Parameters** (Yol Parametreleri) ve **Query Parameters** (Sorgu Parametreleri).

### Path Parameters (URL İçindeki Değişkenler)

Rest API'lerdeki `/users/123` veya `/products/shoe-45/reviews` gibi dinamik ID'leri yakalamak için URL'de iki nokta üst üste `:` kullanırız:

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  // ':id' burada dinamik bir değişkendir
  http.get('https://api.example.com/users/:id', ({ params }) => {
    // 1. Parametreyi al (params her zaman string döner)
    const { id } = params
    
    console.log('İstenen User ID:', id)

    // 2. Logic kurabiliriz: ID'ye göre farklı cevap dön
    if (id === '999') {
      return new HttpResponse(null, { 
        status: 404, 
        statusText: 'User Not Found' 
      })
    }

    // 3. ID'yi cevabın içine gömerek geri dönelim
    return HttpResponse.json({
      id: Number(id),
      name: 'Dinamik Kullanıcı',
      description: `${id} numaralı kullanıcının detayları burada.`
    })
  }),
]
```

#### Iç İçe Path Parametreleri

Daha karmaşık URL yapılarında birden fazla path parametresi kullanabilirsiniz:

```typescript
http.get('https://api.example.com/users/:userId/posts/:postId', ({ params }) => {
  const { userId, postId } = params

  return HttpResponse.json({
    userId,
    postId,
    title: `User ${userId} için Post ${postId}`,
    content: 'Lorem ipsum...'
  })
})
```

### Query Parameters (Soru İşaretinden Sonrakiler)

Query parametreleri (`?page=2&filter=active`), handler URL'ine yazılmaz. Bunları `request` objesinden ayıklarız:

```typescript
http.get('https://api.example.com/products', ({ request }) => {
  // 1. URL objesini oluştur
  const url = new URL(request.url)

  // 2. Parametreleri oku
  const category = url.searchParams.get('category')
  const page = url.searchParams.get('page') || '1'

  // Mock veri havuzu
  const allProducts = [
    { id: 1, name: 'Laptop', category: 'electronics' },
    { id: 2, name: 'T-Shirt', category: 'clothing' },
    { id: 3, name: 'Mouse', category: 'electronics' },
  ]

  // 3. Logic: Kategoriye göre filtrele
  let filteredProducts = allProducts
  
  if (category) {
    filteredProducts = allProducts.filter(p => p.category === category)
  }

  return HttpResponse.json({
    data: filteredProducts,
    meta: {
      currentPage: Number(page),
      total: filteredProducts.length
    }
  })
})
```

#### Query Parametreleriyle İstek Örneği

```typescript
// src/pages/ProductList.tsx
import { useEffect, useState } from 'react'

export const ProductList = () => {
  const [products, setProducts] = useState([])

  useEffect(() => {
    // Handler otomatik olarak 'electronics' kategorisini yakalar
    fetch('https://api.example.com/products?category=electronics&page=1')
      .then(res => res.json())
      .then(data => setProducts(data.data))
  }, [])

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>{product.name}</li>
      ))}
    </ul>
  )
}
```

### TypeScript ile Tip Güvenliği

TypeScript kullanıyorsanız, params objesinin içindeki değerlerin tipini tanımlayarak otomatik tamamlama (autocomplete) desteği alabilirsiniz:

```typescript
// Path parametrelerinin tiplerini tanımlıyoruz
type UserPostParams = {
  userId: string
  postId: string
}

// Handler'a Generic olarak geçiyoruz
http.get<UserPostParams>(
  'https://api.example.com/users/:userId/posts/:postId', 
  ({ params }) => {
    // Artık params.userId ve params.postId string olarak tanınıyor
    const { userId, postId } = params
    
    return HttpResponse.json({
      title: `User ${userId} için Post ${postId} başlığı`,
      content: 'Lorem ipsum...',
      author: `User #${userId}`,
      postNumber: `Post #${postId}`
    })
  }
)
```

## Environment Variables ile Koşullu MSW Aktivasyonu

Bazı durumlarda geliştirme yaparken her zaman mock data kullanmak isteyebileceğiniz bir senaryo olabilir. MSW'yi `.env` dosyasındaki bir flag ile kontrol edebileceğiniz şekilde ayarlayabilirsiniz. Bu sayede, geliştirme ortamında gerçek API'ye bağlanmak istediğiniz zamanlarda MSW'yi devre dışı bırakabilirsiniz.

### Adım 1: .env Dosyası Ayarlaması

Proje kök klasörüne `.env` ve `.env.example` dosyaları oluşturun:

```bash
# .env (Git tarafından ignore edilir)
VITE_MOCK_DATA=true
VITE_API_BASE_URL=https://api.example.com

# .env.example (Version control'e katılır)
VITE_MOCK_DATA=false
VITE_API_BASE_URL=https://api.example.com
```

**Not:** Vite ile çalışıyorsanız environment variables `VITE_` ön eki ile başlamalıdır.

### Adım 2: main.tsx Dosyasını Güncelleme

MSW'yi başlatacak kodu şu şekilde değiştirin:

```typescript
// src/main.tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.css'

async function enableMocking() {
  // Production'da hiçbir şey yapma
  if (import.meta.env.MODE !== 'development') {
    return
  }

  // Environment variable'ı kontrol et
  const mockDataEnabled = import.meta.env.VITE_MOCK_DATA === 'true'
  
  if (!mockDataEnabled) {
    console.log('📡 Gerçek API kullanılıyor (MSW devre dışı)')
    return
  }

  // MSW'yi başlat
  const { worker } = await import('./mocks/browser')

  return worker.start({
    onUnhandledRequest: 'bypass',
  })
}

enableMocking()
  .then(() => {
    ReactDOM.createRoot(document.getElementById('root')!).render(
      <React.StrictMode>
        <App />
      </React.StrictMode>,
    )
  })
  .catch((error) => {
    console.error('Mock servisini başlatırken hata:', error)
    // Hata durumunda yine de uygulamayı başlat
    ReactDOM.createRoot(document.getElementById('root')!).render(
      <React.StrictMode>
        <App />
      </React.StrictMode>,
    )
  })
```

### Adım 3: Geliştirilmiş Handler Yapısı

Gerçek API ile mock API'yi seçilebilir hale getirmek için handlers dosyasında bir kontrol mekanizması ekleyebilirsiniz:

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

type User = {
  id: number
  name: string
  email: string
  role: 'admin' | 'user'
}

// Mock veri havuzu
const mockUsers: User[] = [
  { id: 1, name: 'Ahmet Yılmaz', email: 'ahmet@example.com', role: 'admin' },
  { id: 2, name: 'Ayşe Demir', email: 'ayse@example.com', role: 'user' },
  { id: 3, name: 'Mehmet Kaya', email: 'mehmet@example.com', role: 'user' },
]

export const handlers = [
  // GET - Tüm kullanıcıları listele
  http.get(
    `${import.meta.env.VITE_API_BASE_URL}/users`,
    () => {
      return HttpResponse.json(mockUsers)
    }
  ),

  // GET - Tek kullanıcıyı getir
  http.get(
    `${import.meta.env.VITE_API_BASE_URL}/users/:id`,
    ({ params }) => {
      const user = mockUsers.find(u => u.id === Number(params.id))

      if (!user) {
        return new HttpResponse(null, { status: 404 })
      }

      return HttpResponse.json(user)
    }
  ),

  // POST - Yeni kullanıcı oluştur
  http.post(
    `${import.meta.env.VITE_API_BASE_URL}/users`,
    async ({ request }) => {
      const newUser = await request.json() as Omit<User, 'id'>

      const createdUser: User = {
        id: Math.max(...mockUsers.map(u => u.id)) + 1,
        ...newUser
      }

      mockUsers.push(createdUser)

      return HttpResponse.json(createdUser, { status: 201 })
    }
  ),

  // PUT - Kullanıcıyı güncelle
  http.put(
    `${import.meta.env.VITE_API_BASE_URL}/users/:id`,
    async ({ params, request }) => {
      const userIndex = mockUsers.findIndex(u => u.id === Number(params.id))

      if (userIndex === -1) {
        return new HttpResponse(null, { status: 404 })
      }

      const updatedData = await request.json() as Partial<User>
      mockUsers[userIndex] = { ...mockUsers[userIndex], ...updatedData }

      return HttpResponse.json(mockUsers[userIndex])
    }
  ),

  // DELETE - Kullanıcıyı sil
  http.delete(
    `${import.meta.env.VITE_API_BASE_URL}/users/:id`,
    ({ params }) => {
      const userIndex = mockUsers.findIndex(u => u.id === Number(params.id))

      if (userIndex === -1) {
        return new HttpResponse(null, { status: 404 })
      }

      const deletedUser = mockUsers[userIndex]
      mockUsers.splice(userIndex, 1)

      return HttpResponse.json(deletedUser)
    }
  ),
]
```

### Adım 4: API Service Oluşturma (Gelişmiş Hata Yönetimi)

API çağrılarını merkezi bir yerde yönetmek için bir service dosyası oluşturun. Gelişmiş hata yönetimi ile backend'den gelen detaylı hata mesajlarını da gösterebilirsiniz:

```typescript
// src/services/api.ts

// Hata tipi tanımı
interface ApiError {
  status: number
  message: string
  details?: string
  timestamp?: string
}

class ApiErrorHandler extends Error {
  status: number
  details?: string
  timestamp?: string

  constructor(error: ApiError) {
    super(error.message)
    this.name = 'ApiError'
    this.status = error.status
    this.details = error.details
    this.timestamp = error.timestamp
  }
}

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL

// Hata işleme fonksiyonu
async function handleApiError(response: Response): Promise<never> {
  let errorData: any = {}
  
  try {
    errorData = await response.json()
  } catch (e) {
    // JSON parse hatası durumunda fallback
    errorData = {}
  }

  const error: ApiError = {
    status: response.status,
    message: errorData.message || `HTTP ${response.status}: ${response.statusText}`,
    details: errorData.error || errorData.details,
    timestamp: new Date().toISOString(),
  }

  console.error('🚨 API Error:', error)
  throw new ApiErrorHandler(error)
}

export const apiService = {
  // Kullanıcı listesi
  getUsers: async () => {
    try {
      const response = await fetch(`${API_BASE_URL}/users`)
      
      if (!response.ok) {
        await handleApiError(response)
      }
      
      return response.json()
    } catch (error) {
      if (error instanceof ApiErrorHandler) {
        throw error
      }
      throw new ApiErrorHandler({
        status: 0,
        message: 'Ağ bağlantısı hatası: Kullanıcılar yüklenemedi',
        details: error instanceof Error ? error.message : 'Bilinmeyen hata',
      })
    }
  },

  // Tek kullanıcı
  getUser: async (id: number) => {
    try {
      const response = await fetch(`${API_BASE_URL}/users/${id}`)
      
      if (!response.ok) {
        await handleApiError(response)
      }
      
      return response.json()
    } catch (error) {
      if (error instanceof ApiErrorHandler) {
        throw error
      }
      throw new ApiErrorHandler({
        status: 0,
        message: 'Kullanıcı yüklenemedi',
        details: error instanceof Error ? error.message : 'Bilinmeyen hata',
      })
    }
  },

  // Yeni kullanıcı ekle
  createUser: async (userData: { name: string; email: string; role: string }) => {
    try {
      const response = await fetch(`${API_BASE_URL}/users`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData),
      })
      
      if (!response.ok) {
        await handleApiError(response)
      }
      
      return response.json()
    } catch (error) {
      if (error instanceof ApiErrorHandler) {
        throw error
      }
      throw new ApiErrorHandler({
        status: 0,
        message: 'Kullanıcı oluşturulamadı',
        details: error instanceof Error ? error.message : 'Bilinmeyen hata',
      })
    }
  },

  // Kullanıcı güncelle
  updateUser: async (id: number, userData: Partial<any>) => {
    try {
      const response = await fetch(`${API_BASE_URL}/users/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(userData),
      })
      
      if (!response.ok) {
        await handleApiError(response)
      }
      
      return response.json()
    } catch (error) {
      if (error instanceof ApiErrorHandler) {
        throw error
      }
      throw new ApiErrorHandler({
        status: 0,
        message: 'Kullanıcı güncellenemedi',
        details: error instanceof Error ? error.message : 'Bilinmeyen hata',
      })
    }
  },

  // Kullanıcı sil
  deleteUser: async (id: number) => {
    try {
      const response = await fetch(`${API_BASE_URL}/users/${id}`, {
        method: 'DELETE',
      })
      
      if (!response.ok) {
        await handleApiError(response)
      }
      
      return response.json()
    } catch (error) {
      if (error instanceof ApiErrorHandler) {
        throw error
      }
      throw new ApiErrorHandler({
        status: 0,
        message: 'Kullanıcı silinemedi',
        details: error instanceof Error ? error.message : 'Bilinmeyen hata',
      })
    }
  },
}

export type { ApiError }
export { ApiErrorHandler }
```

### Adım 5: Component'te Kullanım (Gelişmiş Hata Görüntüleme)

```typescript
// src/pages/UserList.tsx
import { useEffect, useState } from 'react'
import { apiService, ApiErrorHandler } from '../services/api'

interface ErrorState {
  message: string
  details?: string
  status?: number
}

export const UserList = () => {
  const [users, setUsers] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<ErrorState | null>(null)

  useEffect(() => {
    const fetchUsers = async () => {
      try {
        setLoading(true)
        setError(null)
        const data = await apiService.getUsers()
        setUsers(data)
      } catch (err) {
        if (err instanceof ApiErrorHandler) {
          // API hatası - detaylı bilgi göster
          setError({
            message: err.message,
            details: err.details,
            status: err.status,
          })
          console.error('API Error Details:', {
            status: err.status,
            message: err.message,
            details: err.details,
            timestamp: err.timestamp,
          })
        } else {
          // Bilinmeyen hata
          setError({
            message: 'Beklenmeyen bir hata oluştu',
            details: err instanceof Error ? err.message : 'Bilinmeyen hata',
          })
        }
      } finally {
        setLoading(false)
      }
    }

    fetchUsers()
  }, [])

  if (loading) return <p>Yükleniyor...</p>
  
  if (error) {
    return (
      <div className="error-container" style={{ padding: '20px', backgroundColor: '#fee', borderRadius: '8px' }}>
        <h3 style={{ color: '#c00', margin: '0 0 10px 0' }}>❌ {error.message}</h3>
        {error.status && <p style={{ fontSize: '14px', color: '#666' }}>HTTP {error.status}</p>}
        {error.details && (
          <p style={{ fontSize: '12px', color: '#999', fontFamily: 'monospace' }}>
            Detay: {error.details}
          </p>
        )}
      </div>
    )
  }

  return (
    <div>
      <h1>Kullanıcı Listesi</h1>
      {users.length === 0 ? (
        <p>Kullanıcı bulunamadı</p>
      ) : (
        <ul>
          {users.map((user: any) => (
            <li key={user.id}>
              {user.name} ({user.email}) - <strong>{user.role}</strong>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

**Hata Durumunda Gösterilen Bilgiler:**

```
❌ HTTP 500: Internal Server Error
HTTP 500
Detay: SQL'e ulaşılamadı
```

### MSW vs Gerçek API Geçişi

Şu komutla MSW'yi etkinleştirip devre dışı bırakabilirsiniz:

```bash
# MSW'yi etkinleştir (mock data kullan)
echo "VITE_MOCK_DATA=true" > .env.local

# MSW'yi devre dışı bırak (gerçek API kullan)
echo "VITE_MOCK_DATA=false" > .env.local
```

**Geliştirme aşamasında:**
- `VITE_MOCK_DATA=true` → Mock veriler ile hızlı prototipleme
- `VITE_MOCK_DATA=false` → Gerçek backend ile entegrasyon testi

**Konsol Çıktısı:**

```
✅ MSW Aktif
[MSW] Mocking enabled

❌ MSW Devre Dışı
📡 Gerçek API kullanılıyor (MSW devre dışı)
```