# Docker Compose Rehberi
Tek bir container yönetmek kolaydır, ancak gerçek dünyada uygulamalar bir orkestra gibidir; birden fazla parçanın (Web, API, Veritabanı, Önbellek) uyum içinde çalışması gerekir.

## 1. Sorun: Manuel Yönetim Sorunu
Bir SaaS projesi düşünün. Bu proje çalışırken muhtemelen şunlara ihtiyaç duyacak:

- **.NET API** (Backend)
- **React Uygulaması** (Frontend)
- **PostgreSQL** (Veritabanı)
- **Redis** (Cache için)

Bunların hepsini tek tek `docker run ...` komutlarıyla başlatmak, ağ ayarlarını yapmak ve birbirine bağlamak tam bir kâbustur.

## 2. Çözüm: Docker Compose
Docker Compose, tüm bu servisleri tek bir dosyada (`docker-compose.yml`) tanımlamanızı ve tek bir komutla hepsini birden ayağa kaldırmanızı sağlar.

## 3. Pratik Örnek: Web + Veritabanı
Projenizin ana dizininde `docker-compose.yml` adında bir dosya oluşturduğunuzu düşünün. İçeriği şöyle görünür:

```yaml
version: '3.8'  # Versiyon

services:
  # 1. Servis: Sizin Web Uygulamanız
  saas-api:
    build: .             # "Bulunduğum klasördeki Dockerfile'ı kullan"
    ports:
      - "5000:80"        # Dışarıdan 5000'e gelen, içeri 80'e gitsin
    depends_on:
      - veritabani       # "Veritabanı açılmadan beni başlatma"

  # 2. Servis: Veritabanı (Hazır Image)
  veritabani:
    image: postgres:15   # Resmi Postgres imajını indir
    environment:
      POSTGRES_PASSWORD: gizlisifre # Ortam değişkenleri (Environment Variables)
    volumes:
      - db-data:/var/lib/postgresql/data # KALICILIK (Aşağıda açıklayacağım)

# Verilerin saklanacağı depo tanımı
volumes:
  db-data:
```
## 4. Kritik Kavram: Volumes (Veri Kalıcılığı)
Docker konteynerleri doğası gereği **geçicidir** (ephemeral). Veritabanı containerini silerseniz veya yeniden başlatırsanız, içindeki veri kaybolur.

**Çözüm (Volumes)**
Yukarıdaki `volumes` satırı şunu der: "Postgres'in verileri yazdığı klasörü (`/var/lib/...`), benim bilgisayarımda güvenli bir alana (`db-data`) bağla."

Containeri silseniz, patlatsanız bile verileriniz (Volume) bilgisayarınızda güvende kalır. Yeni container açtığınızda kaldığı yerden devam eder.

## 5. Docker ile Docker Compose Arasındaki Fark
İkisi arasındaki farkı en net şekilde anlamanız için müzisyen ve orkestra şefi ile analoji ve teknik karşılaştırma yapalım.

**Docker (Tekil Müzisyen)**: Bir gitariste "Şu notayı çal" dersiniz. O da çalar. Eğer hem gitarist hem davulcu hem de piyanist istiyorsanız, hepsine tek tek gidip ne yapacaklarını söylemeniz gerekir. **Teknik Karşılığı:** Tek bir container (uygulamayı) çalıştırmak için kullanılır.

**Docker Compose (Orkestra Şefi ve Nota Kağıdı)**: Elinizde bir nota kağıdı (YAML dosyası) vardır. Şef (Compose), bu kağıda bakar ve "Sen gitar çal, sen davula gir, sen de piyano çal" der. Hepsini aynı anda, birbiriyle uyumlu başlatır. **Teknik Karşılığı:** Birden fazla konteyneri aynı anda, tek bir komutla ve birbirine bağlı şekilde çalıştırmak için kullanılır.

### Örnek
Örneğin n8n ve ngrok kullanarak aradaki farkı anlayalım.

**Eğer Sadece "Docker" Kullansaydık**

İki ayrı terminal penceresi açıp şu uzun komutları tek tek yazmanız ve birbirlerini bulmaları için özel ağ ayarları yapmanız gerekirdi:

Önce bir ağ yarat:
```bash
docker network create benim-agim
```

Sonra n8n'i çalıştır (Ağa bağla):
```bash
docker run --network=benim-agim --name n8n ... n8nio/n8n
```

Sonra ngrok'u çalıştır (Ağa bağla ve n8n'i hedefle):
```bash
docker run --network=benim-agim --name ngrok ... ngrok/ngrok http n8n:5678
```

Bu yöntem yorucudur, hata yapmaya açıktır ve bilgisayarı kapatıp açtığınızda her şeyi baştan yazmanız gerekir.

**Docker Compose Kullandığımızda**
Sadece bir dosya (`docker-compose.yml`) oluşturduk. Bu dosya şunları tarif etti:

- "Bana bir n8n lazım."
- "Bana bir ngrok lazım."
- "Bunları aynı ağa koy, ngrok, n8n'i görebilsin."

Tek komutla (`docker compose up`) tüm sistemi ayağa kaldırdınız.

| Özellik | Docker (CLI) | Docker Compose |
|---------|--------------|----------------|
| Odak Noktası | Tek bir konteyner (Servis). | Çoklu konteyner (Uygulama yığını). |
| Yöntem | İmperatif: "Bunu yap, sonra şunu yap." | Deklaratif: "Benim istediğim sistemin tarifi budur, bunu hazırla." |
| Ağ (Networking) | Konteynerlerin konuşması için ağı manuel yaratmalısınız. | Servisler için otomatik olarak ortak bir ağ yaratır. |
| Kalıcılık | Komutu terminale yazar ve unutursunuz (history'de kaybolur). | Ayarlar .yml dosyasında saklanır, yıllar sonra bile aynı çalışır. |
| Başlatma | `docker run ...` | `docker compose up` |

Sadece hızlıca bir veritabanı deneyecekseniz veya tek bir uygulama (örneğin sadece n8n) açacaksanız Docker komutu yeterlidir.

Ama birden fazla parça (n8n + veritabanı + ngrok vb.) birbiriyle konuşacaksa **Docker Compose şarttır**.