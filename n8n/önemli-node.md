# n8n Önemli Node'lar

n8n workflow'larında sık kullanılan temel node'lar ve kategorileri aşağıda listelenmiştir.

## Tetikleyiciler (Triggers)

- **Schedule Trigger** - Zamanlanmış görevler için
- **Manual Trigger** (eskiden Manual Workflow Trigger) - Manuel başlatma için
- **Webhook** - Dışarıdan gelen istekleri dinlemek için
- **Error Trigger** - Hata yönetimi için
- **n8n Form Trigger** - Form tetikleyicisi için
- **Workflow Trigger** - Başka workflow'lardan tetikleme için

## Veri İşleme ve Manipülasyon

- **Edit Fields (Set)** - Veri alanları ekleme/değiştirme için (eski adı: Set) n8n
- **Code** - JavaScript/Python ile özel mantık için
- **Aggregate** - Veriyi gruplama ve toplama işlemleri için
- **Split Out** - Liste işlemleri için (eski adı: Item Lists)
- **Merge** - Farklı veri akışlarını birleştirmek için
- **Loop Over Items (Split in Batches)** - Büyük veri setlerini parçalara ayırmak için n8n
- **Compare Datasets** - Veri setlerini karşılaştırmak için
- **Remove Duplicates** - Tekrarlanan verileri kaldırmak için
- **Sort** - Veriyi sıralamak için
- **Rename Keys** - Anahtar isimlerini değiştirmek için

## Akış Kontrolü (Flow Control)

- **If** - Koşullu mantık için
- **Switch** - Çoklu koşullu dallanma için
- **Filter** - Veriyi filtrelemek için
- **Wait** - İş akışını duraklatmak için
- **Stop And Error** - İş akışını durdurmak veya hata fırlatmak için n8n
- **Execute Sub-workflow** (eskiden Execute Workflow) - Başka bir workflow'u çalıştırmak için n8n
- **Limit** - Veri akışını sınırlamak için

## Dış Servis Entegrasyonları

- **HTTP Request** - API çağrıları için
- **Google Sheets** - Spreadsheet işlemleri için
- **Gmail** - E-posta gönderme/alma için
- **Google Drive** - Dosya yönetimi için
- **Slack** - Mesajlaşma ve bildirimler için
- **Telegram** - Bot ve bildirimler için
- **Postgres / MySQL** - Veritabanı işlemleri için

## Dosya İşleme

- **Read/Write Files from Disk** - Dosya okuma/yazma için
- **Extract From File** - Dosyalardan veri çıkarma için
- **Convert to File** - Veriyi dosyaya dönüştürme için
- **Edit Image** - Görsel düzenleme için
- **Compression** - Sıkıştırma işlemleri için

## Yardımcı ve Diğer

- **Sticky Note** (eskiden Sticky) - Workflow'a açıklama notu eklemek için
- **n8n Form** - Form oluşturma için
- **Date & Time** - Tarih ve saat işlemleri için
- **Crypto** - Şifreleme işlemleri için
- **HTML** - HTML işleme için
- **Markdown** - Markdown dönüşümleri için
- **XML** - XML işleme için
- **Send Email** - E-posta gönderme için
- **No Operation, do nothing** - Hiçbir işlem yapmayan placeholder node

## AI ve Gelişmiş

- **AI Transform** - AI ile veri dönüştürme için
- **AI Agent** - AI agent'ları için
- **Summarize** - Metin özetleme için
- **Chat Trigger** - Sohbet tetikleyicisi için

## En Önemli Değişiklikler

- **"Set"** → **"Edit Fields (Set)"** oldu
- **"Execute Workflow"** → **"Execute Sub-workflow"** oldu
- **"Split in Batches"** → **"Loop Over Items (Split in Batches)"** oldu
- **"Item Lists"** → **"Split Out"** oldu