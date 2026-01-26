# n8n Expression'ları

n8n expression'ları, n8n workflow otomasyon platformunda dinamik veri işleme ve manipülasyonu için kullanılan JavaScript tabanlı ifadelerdir.

## Temel Özellikleri

n8n'de expression'lar çift süslü parantez içinde yazılır: `{{ }}`. Bu parantezler içinde JavaScript kodu yazabilir, workflow içindeki verilere erişebilir ve bunları dönüştürebilirsiniz.

## Yaygın Kullanım Alanları

- **Veri erişimi:** Önceki node'lardan gelen verilere erişmek için `{{ $json.fieldName }}` veya `{{ $node["NodeName"].json }}`
- **Tarih/saat işlemleri:** `{{ new Date().toISOString() }}` veya `{{ DateTime.now().toFormat('yyyy-MM-dd') }}`
- **String manipülasyonu:** `{{ $json.name.toUpperCase() }}` veya `{{ $json.email.split('@')[0] }}`
- **Matematiksel işlemler:** `{{ $json.price * 1.18 }}` (KDV dahil fiyat hesaplama gibi)
- **Koşullu işlemler:** `{{ $json.status === 'active' ? 'Aktif' : 'Pasif' }}`

## Örnek Kullanım

Diyelim ki bir API'den kullanıcı bilgileri alıyorsunuz ve tam adı oluşturmak istiyorsunuz:

```javascript
{{ $json.firstName + ' ' + $json.lastName }}
```

n8n expression'ları, workflow'larınızı dinamik ve esnek hale getirerek farklı senaryolara göre veri işleme yapmanızı sağlar.

## Veri Erişim Yöntemleri

n8n'de bu üç farklı veri erişim yönteminin detaylı karşılaştırması:

### 1. $('NodeName').item.json
Modern ve önerilen yöntem (n8n v1.0+)

#### Kullanım Senaryoları
```javascript
// Belirli bir node'dan veri çekme
{{ $('HTTP Request').item.json.userId }}

// Birden fazla item varsa ilk item'a erişir
{{ $('Webhook').item.json.name }}

// Nested (iç içe) verilere erişim
{{ $('API Call').item.json.response.data.email }}

// Array içindeki veriye erişim
{{ $('Get Users').item.json.users[0].name }}
```

#### Avantajları
- Daha okunabilir ve modern syntax
- Autocomplete desteği daha iyi
- Node adı değiştiğinde hata mesajları daha açık

### 2. {{ $json.fieldName }}
Mevcut node'un verisine erişim (kısa yol)

#### Kullanım Senaryoları
```javascript
// Set node içinde değer atama
{{ $json.firstName + ' ' + $json.lastName }}

// Code node içinde mevcut veriyi kullanma
{{ $json.price * 1.18 }}

// Function node'da mevcut item'ı işleme
{{ $json.email.toLowerCase() }}

// IF node'da koşul kontrolü
{{ $json.age >= 18 }}

// Aynı node içindeki farklı field'lara referans
{{ $json.quantity * $json.unitPrice }}
```

#### Kısıtlamaları
- Sadece mevcut işlenen item'a erişir
- Başka node'lardaki verilere erişemez
- Loop içinde her iterasyondaki mevcut item'ı temsil eder

### 3. {{ $node["NodeName"].json }}
Eski syntax (legacy, geriye dönük uyumluluk için)

#### Kullanım Senaryoları
```javascript
// Eski workflow'larda görebilirsiniz
{{ $node["Webhook"].json.userId }}

// Dinamik node adı ile erişim (nadir)
{{ $node[$json.nodeName].json.value }}

// Node adında özel karakter varsa
{{ $node["API Call - Production"].json.result }}
```

#### Dezavantajları
- Daha az okunabilir
- Yeni projelerde kullanılması önerilmez
- `$('NodeName')` ile değiştirilmeli

## Karşılaştırmalı Örnekler

### Senaryo 1: E-ticaret Sipariş İşleme
```javascript
// Webhook'tan gelen sipariş
Webhook node → $json.orderId = "ORD-123"

// Set node'da sipariş detayı oluşturma
{
  "orderNumber": "{{ $json.orderId }}", // Mevcut item
  "customer": "{{ $('Webhook').item.json.customerName }}", // Webhook'tan
  "total": "{{ $json.items.reduce((sum, item) => sum + item.price, 0) }}"
}

// HTTP Request node'da API'ye gönderme
URL: https://api.example.com/orders/{{ $('Webhook').item.json.orderId }}
```

### Senaryo 2: Çoklu Node'dan Veri Birleştirme
```javascript
// Node 1: "Get User" → user bilgisi
// Node 2: "Get Orders" → sipariş listesi
// Node 3: "Get Payments" → ödeme bilgisi

// Merge node'da birleştirme
{
  "userName": "{{ $('Get User').item.json.name }}",
  "orderCount": "{{ $('Get Orders').item.json.length }}",
  "totalSpent": "{{ $('Get Payments').item.json.total }}",
  "currentItem": "{{ $json.someField }}" // Merge'den gelen mevcut item
}
```

### Senaryo 3: Loop İçinde İşlem
```javascript
// Split In Batches node ile loop

// Loop içindeki her item için
{{ $json.productName }} // Her iterasyonda farklı ürün

// Loop dışındaki sabit veriye erişim
{{ $('Initial Data').item.json.companyName }} // Hep aynı değer

// Loop sayacı
{{ $json.$index }} // 0, 1, 2, 3...
```

### Senaryo 4: Koşullu İşlemler (IF Node)
```javascript
// IF node koşulları

// Mevcut item kontrolü
{{ $json.status === 'active' }}

// Başka node'dan gelen veri ile karşılaştırma
{{ $json.userId === $('Get Admin').item.json.adminId }}

// Kompleks koşul
{{ $json.price > $('Config').item.json.maxPrice && $json.stock > 0 }}
```

### Senaryo 5: Hata Yönetimi
```javascript
// Try-catch benzeri yapı

// Varsayılan değer kullanma
{{ $json.email || 'no-email@example.com' }}

// Null check
{{ $('API Response').item.json?.data?.user?.name ?? 'Unknown' }}

// Önceki node'dan hata kontrolü
{{ $('HTTP Request').item.json.success ? $json.result : 'Failed' }}
```

## Hangisini Ne Zaman Kullanmalı?

### $json kullan:
- Aynı node içinde field'lar arası işlem yaparken
- Set, Function, Code node'larda
- Mevcut item'ı manipüle ederken

### $('NodeName') kullan:
- Başka node'lardan veri çekerken
- Yeni projeler ve workflow'larda
- Kodun okunabilirliği önemli olduğunda

### $node["NodeName"] kullanma:
- Sadece eski workflow'ları güncellerken göreceksiniz
- Yeni kod yazmayın bu syntax ile

## Pratik İpuçları
```javascript
// ✅ İYİ: Modern ve okunabilir
{{ $('Webhook').item.json.email }}

// ❌ KÖTÜ: Eski syntax
{{ $node["Webhook"].json.email }}

// ✅ İYİ: Mevcut item için kısa yol
{{ $json.firstName }}

// ❌ KÖTÜ: Gereksiz uzun
{{ $('Current Node').item.json.firstName }}

// ✅ İYİ: Multiple items'a erişim
{{ $('Get All').all()[0].json.name }}

// ✅ İYİ: Son item
{{ $('Get All').last().json.status }}
```

## Birden Fazla Item Durumunda Davranış

Birden fazla item durumunda bu methodlar farklı davranır. Detaylı açıklayayım:

### 1. $('NodeName').item.json - Sadece İLK Item'ı Alır ⚠️
```javascript
// "Get Users" node'u 5 kullanıcı döndürüyor

{{ $('Get Users').item.json.name }}
// Sonuç: Sadece ilk kullanıcının adı (örn: "Ahmet")

{{ $('Get Users').item.json.email }}
// Sonuç: Sadece ilk kullanıcının emaili
```

**Problem:** Diğer 4 kullanıcıyı görmezden gelir! ❌

### 2. {{ $json.fieldName }} - Loop'ta Her Item İçin Çalışır ✅
```javascript
// "Get Users" → 5 item döndü
// Code/Set node otomatik olarak her item için çalışır

{{ $json.name }} 
// 1. çalışma: "Ahmet"
// 2. çalışma: "Mehmet"  
// 3. çalışma: "Ayşe"
// ... böyle devam eder
```

n8n otomatik olarak her item için node'u çalıştırır (implicit loop).

## Birden Fazla Item'a Erişim Yöntemleri

### Method 1: .all() - Tüm Item'ları Al
```javascript
// Tüm kullanıcıların adlarını array olarak al
{{ $('Get Users').all().map(item => item.json.name) }}
// Sonuç: ["Ahmet", "Mehmet", "Ayşe", "Fatma", "Ali"]

// Tüm emailler
{{ $('Get Users').all().map(item => item.json.email) }}

// İlk 3 kullanıcı
{{ $('Get Users').all().slice(0, 3).map(item => item.json.name) }}

// Filtreleme: Sadece aktif kullanıcılar
{{ $('Get Users').all().filter(item => item.json.status === 'active') }}
```

### Method 2: .first() ve .last() - İlk/Son Item
```javascript
// İlk item (item ile aynı)
{{ $('Get Users').first().json.name }}

// Son item
{{ $('Get Users').last().json.email }}

// Son 3 item
{{ $('Get Users').all().slice(-3) }}
```

### Method 3: Index ile Erişim [index]
```javascript
// 2. kullanıcı (index 1)
{{ $('Get Users').all()[1].json.name }}

// 5. kullanıcı (index 4)  
{{ $('Get Users').all()[4].json.email }}

// Son kullanıcı
{{ $('Get Users').all()[$('Get Users').all().length - 1].json.name }}
```

### Method 4: .itemMatching() - Koşula Göre Bul
```javascript
// Email'i belirli bir değer olan kullanıcıyı bul
{{ $('Get Users').itemMatching(0).json.name }}
// 0. index'teki item'ı matching kabul eder (kullanımı kısıtlı)

// Daha pratik: JavaScript filter kullan
{{ $('Get Users').all().find(item => item.json.email === 'ahmet@example.com')?.json.name }}
```

## Pratik Örnekler - Gerçek Senaryolar

### Senaryo 1: E-posta Listesi Oluşturma
```javascript
// "Get Customers" → 50 müşteri döndü

// ❌ YANLIŞ: Sadece ilk müşteri
{{ $('Get Customers').item.json.email }}

// ✅ DOĞRU: Tüm emailler (virgülle ayrılmış)
{{ $('Get Customers').all().map(item => item.json.email).join(', ') }}

// ✅ DOĞRU: HTML liste olarak
{{ $('Get Customers').all().map(item => `<li>${item.json.email}</li>`).join('') }}
```

### Senaryo 2: Toplam Hesaplama
```javascript
// "Get Orders" → 20 sipariş döndü

// Toplam sipariş tutarı
{{ $('Get Orders').all().reduce((sum, item) => sum + item.json.total, 0) }}

// Ortalama sipariş değeri
{{ $('Get Orders').all().reduce((sum, item) => sum + item.json.total, 0) / $('Get Orders').all().length }}

// En yüksek sipariş
{{ Math.max(...$('Get Orders').all().map(item => item.json.total)) }}
```

### Senaryo 3: Filtreleme ve Dönüştürme
```javascript
// "Get Products" → 100 ürün döndü

// Sadece stokta olan ürünler
{{ $('Get Products').all().filter(item => item.json.stock > 0) }}

// Fiyatı 100 TL'den yüksek ürünler
{{ $('Get Products').all().filter(item => item.json.price > 100).map(item => item.json.name) }}

// İndirimli ürünlerin toplam değeri
{{ $('Get Products').all()
    .filter(item => item.json.discount > 0)
    .reduce((sum, item) => sum + item.json.price, 0) }}
```

### Senaryo 4: Loop vs. Single Processing
```javascript
// Yöntem A: n8n'in otomatik loop'unu kullan
// Set node her item için ayrı ayrı çalışır
{
  "name": "{{ $json.productName }}", // Her üründe farklı
  "discounted": "{{ $json.price * 0.8 }}" // Her ürün için hesapla
}
// Çıktı: 100 item → 100 item (her biri işlenmiş)

// Yöntem B: Tek bir item'da tüm veriyi topla
// Code node veya Function node
{
  "allProducts": {{ $('Get Products').all().map(item => ({
    name: item.json.productName,
    discounted: item.json.price * 0.8
  })) }}
}
// Çıktı: 100 item → 1 item (içinde 100 elemanlı array)
```

### Senaryo 5: Belirli Sıradaki Item'ı Kullanma
```javascript
// "Get API Responses" → 10 response döndü

// 1. response (en güncel)
{{ $('Get API Responses').first().json.data }}

// Son response (en eski)
{{ $('Get API Responses').last().json.data }}

// 5. response
{{ $('Get API Responses').all()[4].json.data }}

// Her 2. item'ı al (0, 2, 4, 6...)
{{ $('Get API Responses').all().filter((item, index) => index % 2 === 0) }}
```

### Senaryo 6: Merge ile Birleştirme
```javascript
// "Get Users" → 5 kullanıcı
// "Get Orders" → 20 sipariş
// Her kullanıcı için siparişlerini eşleştir

// Merge node (Mode: Keep Matches)
{
  "userName": "{{ $json.name }}", // Mevcut user
  "userOrders": {{ $('Get Orders').all()
    .filter(order => order.json.userId === $json.id)
    .map(order => order.json.orderNumber) }}
}
```

### Senaryo 7: Split In Batches ile İşleme
```javascript
// "Split In Batches" → Her seferinde 10 item işle

// Batch içindeki tüm item'lar
{{ $input.all().map(item => item.json.email) }}

// Batch numarası
{{ $node["Split In Batches"].context.noItemsLeft ? 'Son Batch' : 'Devam Ediyor' }}

// İşlenen toplam item sayısı
{{ $input.all().length }}
```

## Hata Önleme İpuçları
```javascript
// ✅ Güvenli: Item var mı kontrol et
{{ $('Get Users').all().length > 0 ? $('Get Users').first().json.name : 'Kullanıcı yok' }}

// ✅ Güvenli: Optional chaining
{{ $('API Call').item.json?.response?.data?.user?.name ?? 'Bilinmiyor' }}

// ❌ Tehlikeli: Item yoksa hata verir
{{ $('Get Users').item.json.name }}

// ✅ Güvenli: Varsayılan değer
{{ $('Get Users').item.json.name || 'İsimsiz' }}
```

## Özet Tablo

| Method | Tek Item | Çoklu Item | Kullanım |
|--------|----------|------------|----------|
| .item.json | ✅ Alır | ⚠️ Sadece ilki | Tek item beklediğinde |
| .first().json | ✅ Alır | ✅ İlki | İlk item gerektiğinde |
| .last().json | ✅ Alır | ✅ Sonuncu | Son item gerektiğinde |
| .all() | ✅ Array[1] | ✅ Array[n] | Tümünü işlerken |
| .all()[i] | ✅ Index 0 | ✅ i'nci item | Belirli sıradaki |
| $json | ✅ Mevcut | ✅ Loop'ta her biri | Otomatik loop |