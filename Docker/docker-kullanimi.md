# Docker Kullanım Rehberi

> **İçindekiler Tablosu**
> - [Temel Komutlar](#temel-komutlar)
> - [Container Yönetimi](#container-yönetimi)
> - [Parametre Açıklamaları](#parametre-açıklamaları)
> - [docker exec Kullanımı](#docker-exec-kullanımı)
> - [Özet Tabloları](#özet-tabloları)

---

## Temel Komutlar

### Kurulum Doğrulama
```bash
docker --version
```

### Docker Hub'dan Image İndirme
```bash
docker pull ubuntu
docker pull nginx
docker pull python:3.9
```

### Image Listeleme ve Yönetim

#### Yüklü Image'ları Listeleme
```bash
docker images
```

#### Image Detaylarını Görme
```bash
docker inspect ubuntu
```

#### Image Silme
```bash
docker rmi ubuntu
```

---

## Container Yönetimi

### Container Oluşturma ve Çalıştırma

#### Basit Container (Tek Komut)
```bash
docker run ubuntu echo "Merhaba Docker"
```

#### İnteraktif Terminal
```bash
docker run -it ubuntu /bin/bash
```

#### Arka Planda Çalışan Container
```bash
docker run -d --name webserver nginx
```

#### Port Yönlendirmesi (Host:Container)
```bash
docker run -d -p 8080:80 --name web nginx
```

#### Environment Variable ile
```bash
docker run -d -e DATABASE_URL=localhost --name app python:3.9
```

### Container Listeleme ve İzleme

#### Çalışan Container'ları Listeleme
```bash
docker ps
```

#### Tüm Container'ları Listeleme (Durdurulmuş Dahil)
```bash
docker ps -a
```

### Container Kontrol Komutları

#### Container Durdurma/Başlatma/Silme
```bash
# Durdurma
docker stop webserver

# Başlatma  
docker start webserver

# Silme (önce durdurmalısınız)
docker stop webserver
docker rm webserver
```

#### Log Yönetimi
```bash
# Logları görme
docker logs webserver

# Logları canlı takip etme
docker logs -f webserver
```

---

## Parametre Açıklamaları

### İnteraktif Terminal (-it)

**Ne yapar?** Ubuntu container'ının içine doğrudan girmenizi sağlar.

```bash
docker run -it ubuntu /bin/bash
```

**Parametreler:**

- **`-i` (interactive):** Container'ın giriş akışını açık tutup sizin yazı yazmanızı sağlar
- **`-t` (tty):** Terminal oluşturur, böylece komut yazabilirsiniz  
- **`/bin/bash`:** Container'da bash shell'ini çalıştırır

**Pratik Örnek:**
```bash
docker run -it ubuntu /bin/bash
# Artık container içindeyiz:
root@a1b2c3d4e5f6:/#

# Container içinde komutlar çalıştırabiliriz
root@a1b2c3d4e5f6:/# ls
root@a1b2c3d4e5f6:/# pwd  
root@a1b2c3d4e5f6:/# apt-get update

# Container'dan çıkış
root@a1b2c3d4e5f6:/# exit
```

**Fark:**
```bash
# ✅ İnteraktif - terminal açar
docker run -it ubuntu /bin/bash

# ❌ İnteraktif olmayan - sadece komut çalıştırır
docker run ubuntu echo "Merhaba"
```

### Detached Mode (-d)

**Ne yapar?** Container'ı arka planda çalıştırır, terminalinizi bloke etmez.

`-d` = **detached** (arka plan) modu

**Örnekler:**
```bash
# ❌ -d OLMADAN (ön plan - terminal meşgul)
docker run nginx
# Terminal bloke olur, loglar ekrana akar

# ✅ -d İLE (arka plan - terminal serbest)
docker run -d nginx
# Container ID döner: 6f8c9d2e1a4b5c7f
# Terminal başka komutları kabul eder
```

**Pratik Kullanım:**
```bash
# Web sunucusunu arka planda başlat
docker run -d --name webserver nginx

# Terminal serbest - başka işlemler yapabiliriz
docker ps
docker logs webserver
docker stop webserver
```

> **Analoji:** Kahve dükkanı gibi düşünün:
> - **-d olmadan:** Barista sadece sizle uğraşır (terminal meşgul)
> - **-d ile:** Barista siparişi alır, arka planda hazırlar (terminal serbest)

### Follow Mode (-f) 

**Ne yapar?** Logları canlı olarak izlemenizi sağlar.

`-f` = **follow** (takip et)

**Karşılaştırma:**
```bash
# ❌ -f OLMADAN (anlık görüntü)
docker logs webserver
# Şu ana kadarki logları gösterir ve çıkar

# ✅ -f İLE (canlı takip)  
docker logs -f webserver
# Mevcut logları gösterir ve yeni gelenleri bekler
# Ctrl+C ile çıkabilirsiniz
```

**Pratik Senaryo:**
```bash
# Terminal 1: Web sunucusu başlat
docker run -d --name webserver nginx

# Terminal 2: Logları canlı takip et
docker logs -f webserver

# Terminal 3: İstekler gönder  
# http://localhost:8080 

# Terminal 2'de anlık logları göreceksiniz!
```

---

## Container İsimlendirme (--name)

**Neden gerekli?** Container'ları karışık ID'ler yerine anlamlı isimlerle yönetmek.

### Karşılaştırma

**❌ İsim vermeden:**
```bash
docker run -d nginx
# Docker rastgele ID atar: 6f8c9d2e1a4b5c7f9a2b3c4d5e6f7g8h

# Yönetirken zorlanırsınız:
docker stop 6f8c9d2e1a4b5c7f9a2b3c4d5e6f7g8h
```

**✅ İsim vererek:**
```bash
docker run -d --name webserver nginx

# Kolay yönetim:
docker stop webserver
docker logs webserver  
docker start webserver
```

### Pratik Örnekler

```bash
# Farklı servisler için anlamlı isimler
docker run -d --name webserver nginx
docker run -d --name database postgres
docker run -d --name api python:3.9

# İsimlerle kolay yönetim
docker ps  # NAMES sütununda: webserver, database, api
docker stop webserver
docker logs database
docker restart api
```

### Önemli Kurallar

```bash
# ❌ Aynı isim kullanılamaz
docker run -d --name web nginx
docker run -d --name web nginx  # HATA: name already exists

# ✅ Farklı isimler kullanın
docker run -d --name web1 nginx
docker run -d --name web2 nginx
```

---

## docker exec Kullanımı

**Ne yapar?** Çalışan bir container'ın içinde komut çalıştırır.

### docker run vs docker exec

| Özellik | `docker run` | `docker exec` |
|---------|--------------|---------------|
| **Container** | YENİ oluşturur | MEVCUT kullanır |
| **Durum** | Container başlatır/durdurur | Container çalışmaya devam eder |
| **Kullanım** | İlk kez başlatma | Çalışan container'a müdahale |

**📝 Örnek:**
```bash
# docker run - YENİ container oluşturur
docker run ubuntu echo "Merhaba"
# ↳ Yeni container → komut çalışır → container durur

# docker exec - VAROLAN container kullanır  
docker exec mycontainer echo "Merhaba"
# ↳ Çalışan container bulur → komut çalışır → container devam eder
```

### Pratik Kullanım Senaryoları

#### 1. Veritabanı İşlemleri
```bash
# PostgreSQL container'ını başlat
docker run -d \
  --name mydb \
  -e POSTGRES_PASSWORD=secret \
  postgres:13

# Container içinde SQL komutları çalıştır
docker exec mydb psql -U postgres -c "CREATE DATABASE myapp;"
docker exec mydb psql -U postgres -c "CREATE USER newuser;"
```

#### 2. Container'a Terminal ile Giriş  
```bash
# Container'ın içine gir
docker exec -it mydb bash

# Şimdi container içindeyiz
root@container:/# ls
root@container:/# pwd
root@container:/# exit  # Çıkış
```

#### 3. Dosya ve Log İnceleme
```bash
# Dosya içeriğini görüntüle
docker exec myapp cat /app/config.json

# Dosya listesi  
docker exec myapp ls -la /app

# Log dosyasını incele
docker exec myapp tail -f /var/log/app.log
```

#### 4. Python Script Çalıştırma
```bash
# Python container'da script çalıştır
docker exec myapp python /app/script.py

# Doğrudan Python kodu çalıştır
docker exec myapp python -c "print('Container çalışıyor!')"
```

### Gerçek Dünya Senaryosu

```bash
# 1. Servisleri başlat
docker run -d --name db postgres:13
docker run -d --name app myapp:latest  

# 2. Uygulama çalışırken veritabanı ayarları
docker exec db psql -U postgres -c "CREATE DATABASE production;"

# 3. Uygulama debug (çalışırken)
docker exec -it app bash
root@container:/# cat /var/log/error.log

# 4. Canlı veri kontrol
docker exec db psql -U postgres -d production -c "SELECT COUNT(*) FROM users;"
```

---

## Özet Tabloları

### Parametre Referansı

| Parametre | Açıklama | Örnek |
|-----------|----------|-------|
| **`-i`** | Input akışını açık tut | Container'a yazı yazabilirsiniz |
| **`-t`** | Terminal oluştur | Prompt görebilirsiniz |
| **`-it`** | İnteraktif terminal | Container'a doğrudan girersiniz |
| **`-d`** | Arka planda çalıştır | Terminal serbest kalır |
| **`-f`** | Logları canlı takip | Yeni loglar otomatik görünür |
| **`--name`** | Container'a isim ver | Kolay yönetim için |
| **`-p`** | Port yönlendir | `8080:80` → host:container |
| **`-e`** | Environment variable | `DATABASE_URL=localhost` |

### Komut Karşılaştırması  

| Görev | Komut Tipi | Örnek |
|-------|------------|-------|
| **Yeni container** | `docker run` | `docker run -d --name web nginx` |
| **Mevcut container'a komut** | `docker exec` | `docker exec web nginx -s reload` |
| **Container'a giriş** | `docker exec -it` | `docker exec -it web bash` |
| **Log takibi** | `docker logs -f` | `docker logs -f web` |
| **Container durdur** | `docker stop` | `docker stop web` |

### Hızlı Başvuru

```bash
# En Sık Kullanılanlar
docker run -d --name myapp nginx                    # Arka plan başlatma
docker exec -it myapp bash                          # Container'a giriş  
docker logs -f myapp                                # Canlı log takibi
docker stop myapp && docker rm myapp               # Durdur ve sil
docker ps -a                                        # Tüm container'ları listele

# Bilgi Toplama
docker images                                       # Image listesi
docker inspect myapp                                # Container detayları
docker stats                                        # Resource kullanımı
```

---

**Sonuç:** Bu rehber ile Docker container'ları etkili bir şekilde oluşturup yönetebilirsiniz. Her komutun amacını anlamak, Docker'ı daha verimli kullanmanızı sağlar.