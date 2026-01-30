# Azure Blob Storage ile Güvenli Dosya Yönetimi

**İçindekiler**
- [Azure Blob Storage ile Güvenli Dosya Yönetimi](#azure-blob-storage-ile-güvenli-dosya-yönetimi)
  - [Bölüm 1: Mimari, Güvenlik ve Valet Key Deseni](#bölüm-1-mimari-güvenlik-ve-valet-key-deseni)
    - [1. Azure Blob Storage Hiyerarşisi](#1-azure-blob-storage-hiyerarşisi)
    - [2. Mimari Yaklaşım: Valet Key Pattern (Vale Anahtarı Deseni)](#2-mimari-yaklaşım-valet-key-pattern-vale-anahtarı-deseni)
    - [3. SAS (Shared Access Signature) Mantığı](#3-sas-shared-access-signature-mantığı)
    - [4. Uygulayacağımız İş Akışı (Workflow)](#4-uygulayacağımız-i̇ş-akışı-workflow)
  - [Bölüm 2: Azure Portal Yapılandırması ve CORS Ayarları](#bölüm-2-azure-portal-yapılandırması-ve-cors-ayarları)
    - [1. Storage Account (Depolama Hesabı) Oluşturma](#1-storage-account-depolama-hesabı-oluşturma)
    - [2. Container (Kapsayıcı) Oluşturma](#2-container-kapsayıcı-oluşturma)
    - [3. CORS (Cross-Origin Resource Sharing) Ayarları](#3-cors-cross-origin-resource-sharing-ayarları)
    - [4. Bağlantı Bilgilerini (Connection String) Alma](#4-bağlantı-bilgilerini-connection-string-alma)
  - [Bölüm 3: Backend (.NET Core) Katmanı](#bölüm-3-backend-net-core-katmanı)
    - [1. Gerekli Paketlerin Kurulumu](#1-gerekli-paketlerin-kurulumu)
    - [2. Bağlantı Ayarları (appsettings.json)](#2-bağlantı-ayarları-appsettingsjson)
    - [3. Blob Servisi (SAS Token Üreticisi)](#3-blob-servisi-sas-token-üreticisi)
    - [4. API Controller Oluşturma](#4-api-controller-oluşturma)
    - [5. Program.cs (Dependency Injection Kaydı)](#5-programcs-dependency-injection-kaydı)
  - [Bölüm 4: Frontend (React) Katmanı](#bölüm-4-frontend-react-katmanı)
    - [1. Gerekli Paketin Kurulumu](#1-gerekli-paketin-kurulumu)
    - [2. Dosya Yükleme Bileşeni (FileUpload.jsx)](#2-dosya-yükleme-bileşeni-fileuploadjsx)
  - [Bölüm 5: Final Entegrasyon ve Veritabanı Kaydı](#bölüm-5-final-entegrasyon-ve-veritabanı-kaydı)
    - [1. Veritabanı Modeli (Backend)](#1-veritabanı-modeli-backend)
    - [2. API Güncellemesi (Save \& List)](#2-api-güncellemesi-save--list)
    - [3. Frontend Güncellemesi (FileUpload.jsx)](#3-frontend-güncellemesi-fileuploadjsx)
    - [4. Dosyaları Listeleme ve İndirme (FileList.jsx)](#4-dosyaları-listeleme-ve-i̇ndirme-filelistjsx)
  - [Ekstra önlemler](#ekstra-önlemler)

## Bölüm 1: Mimari, Güvenlik ve Valet Key Deseni
### 1. Azure Blob Storage Hiyerarşisi
Kodlamaya geçmeden önce Azure'un depolama mantığını anlamamız gerekir. Azure Storage üç temel katmandan oluşur:
1. **Storage Account (Depolama Hesabı):** Azure'daki en üst düzey yönetim birimidir. Tüm dosyalarınız bu hesabın altında yaşar. Erişim anahtarları (Access Keys) bu seviyede tanımlanır.
2. **Container (Kapsayıcı):** Dosya sistemindeki "Klasör" mantığına benzer. Blobları (dosyaları) gruplamak için kullanılır. İzinler genellikle bu seviyede yönetilir (Örn: "Faturalar" container'ı, "ProfilResimleri" container'ı).
3. **Blob (Binary Large Object):** Dosyanın kendisidir (PDF, JPG, MP4 vb.). Her blob bir container içinde olmalıdır.

### 2. Mimari Yaklaşım: Valet Key Pattern (Vale Anahtarı Deseni)
Geleneksel sistemlerde dosyalar önce Backend sunucusuna, oradan depolama alanına giderdi. Ancak biz bu projede Valet Key Pattern kullanacağız.
**Neden "Vale Anahtarı"?** Arabanızı bir valeye verdiğinizde, ona tüm anahtarlığınızı (ev anahtarı, ofis anahtarı) vermezsiniz. Sadece arabayı çalıştıran ve kapıyı açan, sınırlı yetkiye sahip bir anahtar (veya kart) verirsiniz.

Azure'daki karşılığı şudur:
* **Master Key (Ana Anahtar):** Backend sunucunuzda durur. Her şeyi yapabilir (Silme, yaratma, okuma). Asla Client'a verilmez.
* **Valet Key (SAS Token):** Backend tarafından üretilir. Sadece belirli bir dosya için, belirli bir süre (örn: 10 dk) geçerli olan kısıtlı anahtardır. Client'a bu verilir.

Bu Mimarinin Avantajları:
* Sıfır Sunucu Yükü: Dosya transferi (Upload/Download) sunucu CPU/RAM'ini kullanmaz.
* Düşük Gecikme: Veri doğrudan Client -> Azure arasında akar.
* Güvenlik: Token süresi dolduğunda erişim otomatik olarak kapanır.

### 3. SAS (Shared Access Signature) Mantığı
SAS, depolama kaynaklarınıza sınırlı erişim sağlayan bir URI'dir (Link). Bir SAS Token şunları içerir:

* **Hedef:** Hangi kaynağa erişilecek? (Sadece images container'ı içindeki logo.png dosyası).
* **İzinler (Permissions):** Ne yapılabilir? (Sadece Write - Yazma yetkisi. Okuyamaz veya silemez).
* **Süre (Time-To-Live):** Ne kadar geçerli? (Örn: Start: 14:00, End: 14:15).
* **IP Kısıtlaması:** İsteğe bağlı olarak sadece belirli IP'lerden erişim izni.

Örnek bir SAS URL Yapısı:

```Plaintext
https://mystorage.blob.core.windows.net/docs/fatura.pdf?
sp=w                <-- Permission: Write (Sadece Yazma)
&st=2023-10-01T...  <-- Start Time
&se=2023-10-01T...  <-- Expiry Time (Bitiş)
&spr=https          <-- Protocol (Sadece HTTPS)
&sv=2022-11-02      <-- Service Version
&sr=b               <-- Resource: Blob (Sadece bu dosya)
&sig=d34dB33f...    <-- Signature (İmza - Güvenlik kodu)
```
### 4. Uygulayacağımız İş Akışı (Workflow)
Projemizde React ve .NET Core şu sırayla haberleşecek:

1. Talep (React): Kullanıcı dosya seçer. React, API'ye POST /api/storage/upload-request isteği atar. "Dosya adı: manual.pdf, Boyut: 5MB".
2. Yetkilendirme (API): .NET Core, kullanıcının oturumunu ve yetkisini kontrol eder.
3. Token Üretimi (API): API, Azure SDK kullanarak sadece o dosya ismine özel, 10 dakika geçerli, "Write-Only" (Sadece Yazma) bir SAS Token üretir ve React'a döner.
4. Yükleme (React): React, aldığı bu güvenli URL'i kullanarak dosyayı direkt Azure'a PUT eder.
5. Teyit (React): Yükleme %100 olunca, React API'ye tekrar döner: "Yükleme bitti, veritabanına kaydedebilirsin."

## Bölüm 2: Azure Portal Yapılandırması ve CORS Ayarları
Bu bölümde, Azure üzerinde Storage hesabımızı oluşturacak, dosyaların tutulacağı kutuları (Containers) hazırlayacak ve en önemlisi React uygulamamızın Azure ile konuşabilmesi için CORS (Cross-Origin Resource Sharing) izinlerini açacağız.

### 1. Storage Account (Depolama Hesabı) Oluşturma
Henüz bir hesabınız yoksa, şu adımlarla standart bir hesap oluşturun:

1. Azure Portal'a giriş yapın ve arama çubuğuna "Storage accounts" yazın.
2. **+ Create** butonuna basın.
2. Basics sekmesinde:
   * **Resource Group:** Projeniz için yeni bir grup oluşturun (örn: DMS-Project-RG).
   * **Storage account name:** Benzersiz bir isim verin (örn: firmadmsstorage).
   * **Region:** Kullanıcılarınıza en yakın bölgeyi seçin (Türkiye için genellikle West Europe veya North Europe tercih edilir).
   * **Performance:** Standard (Dökümanlar ve resimler için yeterlidir).
   * **Redundancy:** Başlangıç ve maliyet tasarrufu için Locally-redundant storage (LRS) seçebilirsiniz.
   * **Review + create** diyerek hesabı oluşturun.

### 2. Container (Kapsayıcı) Oluşturma
Dosyalarımızı kategorize etmek için "Container"lar oluşturacağız.
1. Oluşturduğunuz Storage hesabına gidin.
2. Sol menüde Data storage altında Containers'a tıklayın.
3. **+ Container** butonuna basın.
4. Name: company-docs (veya uploads) yazın.
5. Public access level: Burası çok kritiktir.
   * Güvenlik için Private (no anonymous access) seçin.
   * Neden? Çünkü biz dosyalara herkesin erişmesini istemiyoruz. Sadece SAS Token'a sahip olan (yetkili) kişiler erişebilmeli.
6. Create diyerek tamamlayın.

### 3. CORS (Cross-Origin Resource Sharing) Ayarları
Tarayıcı güvenliği gereği, React uygulamanız (localhost:3000) varsayılan olarak Azure (blob.core.windows.net) adresine istek atamaz. Bunu engellemek için CORS kurallarını tanımlamalıyız.

Bu adımı atlamanız durumunda React konsolunda kırmızı renkli "Access to fetch has been blocked by CORS policy" hatası alırsınız.
1. Storage hesabınızın sol menüsünde Settings altında Resource sharing (CORS) seçeneğine tıklayın.
2. Blob service sekmesinde olduğunuzdan emin olun.
3. Aşağıdaki ayarları girin:

| Ayar            | Değer                   | Açıklama                                                                                                                                                         |
| --------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Allowed origins | http://localhost:3000   | React uygulamanızın çalıştığı adres. Canlıya alırken production domaininizi de (örn: https://firma.com) eklemelisiniz. Test için geçici olarak * yapabilirsiniz. |
| Allowed methods | GET, PUT, OPTIONS, HEAD | PUT: Yükleme için. GET: İndirme/Görüntüleme için. OPTIONS: Tarayıcının ön kontrolü (Preflight) için şarttır. |
| Allowed headers | * | Tüm başlıkları kabul et. |
| Exposed headers | * | React'ın Azure'dan gelen yanıt başlıklarını (örn: ETag) okuyabilmesi için. |
| Max age (seconds) | 86400 | Tarayıcının CORS önbellek süresi (24 saat). |

4. Save butonuna basarak ayarları kaydedin.

### 4. Bağlantı Bilgilerini (Connection String) Alma
Backend (.NET Core) tarafının Azure ile konuşabilmesi ve "Vale Anahtarı" (SAS Token) üretebilmesi için ana yönetici şifresine ihtiyacı vardır.
1. Sol menüde **Security + networking** altında **Access keys**'e tıklayın.
2. **key1** altındaki **Connection string** değerini bulun.
3. **Show** diyip kopyalayın ve bir kenara not edin.
   * Örnek: DefaultEndpointsProtocol=https;AccountName=firmadmsstorage;AccountKey=...==;EndpointSuffix=core.windows.net

Şu an Azure tarafında: Depomuz hazır. Kutumuz (Container) kilitli (Private) bir şekilde hazır. React'ın kapıyı çalabilmesi için CORS izni verildi. Backend için ana anahtar (Connection String) alındı.

## Bölüm 3: Backend (.NET Core) Katmanı
### 1. Gerekli Paketlerin Kurulumu
Öncelikle .NET Core Web API projenize Azure Storage kütüphanesini eklemeniz gerekiyor. Terminal veya Package Manager Console üzerinden şu komutu çalıştırın:
```bash
dotnet add package Azure.Storage.Blobs
```
### 2. Bağlantı Ayarları (appsettings.json)
Bölüm 2'de kopyaladığınız Connection String'i projenize tanıtın.
```json
{
  "AzureStorage": {
    "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=firmadmsstorage;AccountKey=...==;EndpointSuffix=core.windows.net",
    "ContainerName": "company-docs"
  }
}
```
### 3. Blob Servisi (SAS Token Üreticisi)
Şimdi token üretim mantığını içeren servisimizi yazalım. Bu servis, Azure SDK'sını kullanarak kriptografik bir imza oluşturacak.

Yeni bir sınıf oluşturun: `Services/BlobService.cs`

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Sas; // SAS işlemleri için gerekli
using Azure.Storage.Blobs.Specialized; // BlockBlobClient için

public class BlobService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly string _containerName;

    public BlobService(IConfiguration configuration)
    {
        // Appsettings'den bilgileri al
        string connectionString = configuration["AzureStorage:ConnectionString"];
        _containerName = configuration["AzureStorage:ContainerName"];

        // Client'ı başlat
        _blobServiceClient = new BlobServiceClient(connectionString);
    }

    // 1. Yükleme (Upload) için Token Üreten Metot
    public string GenerateSasTokenForUpload(string fileName)
    {
        // İlgili container'ı seç
        var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
        
        // O container içinde, bu isimde bir blob (dosya) hedefle
        var blobClient = containerClient.GetBlobClient(fileName);

        // SAS Token Ayarları
        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = _containerName,
            BlobName = fileName,
            Resource = "b", // "b" = Blob (Sadece dosya), "c" = Container (Tüm klasör)
            ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(10) // Token 10 dk sonra ölür
        };

        // YETKİLER: Sadece Yazma (Write) izni veriyoruz.
        // Kullanıcı bu linkle dosyayı okuyamaz veya silemez.
        sasBuilder.SetPermissions(BlobSasPermissions.Write);

        // İmzala ve URI oluştur
        // CanGenerateSasUri özelliği, ConnectionString ile bağlandıysanız True döner.
        Uri sasUri = blobClient.GenerateSasUri(sasBuilder);

        return sasUri.ToString(); // Örn: https://hesap.blob.../dosya.pdf?sig=...
    }

    // 2. İndirme (Download/View) için Token Üreten Metot
    public string GenerateSasTokenForRead(string fileName)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(_containerName);
        var blobClient = containerClient.GetBlobClient(fileName);

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = _containerName,
            BlobName = fileName,
            Resource = "b",
            ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(5) // İndirme linki 5 dk geçerli
        };

        // YETKİLER: Sadece Okuma (Read)
        sasBuilder.SetPermissions(BlobSasPermissions.Read);

        return blobClient.GenerateSasUri(sasBuilder).ToString();
    }
}
```

### 4. API Controller Oluşturma
Şimdi bu servisi dış dünyaya açalım. React, dosya yüklemeden önce buraya istek atacak.

`Controllers/FilesController.cs:`
```csharp
using Microsoft.AspNetCore.Mvc;

[Route("api/[controller]")]
[ApiController]
public class FilesController : ControllerBase
{
    private readonly BlobService _blobService;

    public FilesController(BlobService blobService) // Dependency Injection
    {
        _blobService = blobService;
    }

    // POST: api/files/upload-request
    // React'tan gelen istek: { "fileName": "fatura.pdf" }
    [HttpPost("upload-request")]
    public IActionResult GetUploadUrl([FromBody] UploadRequestDto request)
    {
        // 1. Güvenlik: Kullanıcının dosya yükleme yetkisi var mı? (JWT kontrolü vb.)
        // if (!User.Identity.IsAuthenticated) return Unauthorized();

        // 2. Dosya ismini benzersiz yap (Opsiyonel ama önerilir)
        // Aynı isimde dosya yüklenirse eskisinin üzerine yazar. 
        // Bunu önlemek için Guid ekliyoruz.
        string uniqueFileName = $"{Guid.NewGuid()}_{request.FileName}";

        // 3. SAS Token (URL) üret
        string signedUrl = _blobService.GenerateSasTokenForUpload(uniqueFileName);

        // 4. React'a dön: "Al, bu linke yükle, işlem bitince dosya adını bana bildir"
        return Ok(new 
        { 
            sasUrl = signedUrl, 
            uniqueName = uniqueFileName 
        });
    }
}

// DTO (Data Transfer Object)
public class UploadRequestDto
{
    public string FileName { get; set; }
}
```

### 5. Program.cs (Dependency Injection Kaydı)
Son olarak BlobService'i uygulamaya tanıtmayı unutmayın.

`Program.cs:`
```csharp
// ... diğer servisler
builder.Services.AddScoped<BlobService>();
// ...
```

**Backend Kısmı Tamamlandı!**

Şu an elimizde ne var?
1. **Güvenli bir kapı:** GenerateSasTokenForUpload metodu, Azure'un ana anahtarını kullanarak geçici ve kısıtlı (sadece yazma yetkili) bir link üretiyor.
2. **API:** React uygulaması /api/files/upload-request adresine dosya adını gönderdiğinde, karşılığında yükleme yapabileceği güvenli bir Azure linki alıyor.

## Bölüm 4: Frontend (React) Katmanı
Bu bölümde, büyük dosyaları bile tarayıcıyı dondurmadan yükleyebilen akıllı bir bileşen (component) oluşturacağız.

### 1. Gerekli Paketin Kurulumu
React projenizin olduğu dizinde terminali açın ve resmi Azure Storage SDK'sını kurun:
```bash
npm install @azure/storage-blob
```
> (Not: HTTP istekleri için projenizde axios olduğunu varsayıyorum. Yoksa npm install axios ile ekleyin.)

### 2. Dosya Yükleme Bileşeni (FileUpload.jsx)
Aşağıdaki kod, hem Backend ile konuşup izin alma işlemini hem de Azure'a yükleme işlemini tek bir akışta yapar.

Bu kodu `components/FileUpload.jsx` olarak kaydedin:
```jsx
import React, { useState } from 'react';
import { BlockBlobClient } from '@azure/storage-blob'; // Azure SDK
import axios from 'axios';

const FileUpload = () => {
  const [file, setFile] = useState(null);
  const [progress, setProgress] = useState(0);
  const [status, setStatus] = useState(''); // 'loading', 'success', 'error'

  // 1. Dosya Seçimi
  const onFileChange = (e) => {
    setFile(e.target.files[0]);
    setStatus('');
    setProgress(0);
  };

  // 2. Yükleme İşlemi
  const handleUpload = async () => {
    if (!file) return;

    try {
      setStatus('loading');
      setStatus('Backend\'den izin alınıyor...');

      // ADIM A: Backend'den SAS Token (Yükleme Linki) İste
      // Bu istekte sadece dosya adını gönderiyoruz, dosyanın kendisini değil!
      const response = await axios.post('http://localhost:5000/api/files/upload-request', {
        fileName: file.name
      });

      const { sasUrl, uniqueName } = response.data;
      // sasUrl: https://hesap.blob.../dosya.pdf?sig=... (Yazma izni olan link)

      setStatus('Azure\'a yükleniyor...');

      // ADIM B: Azure SDK'yı Başlat
      // SAS URL'i direkt BlockBlobClient'a veriyoruz.
      const blobClient = new BlockBlobClient(sasUrl);

      // ADIM C: Dosyayı Yükle (Direct Upload)
      // uploadData metodu, dosya büyükse otomatik olarak parçalara (chunk) böler.
      await blobClient.uploadData(file, {
        blobHTTPHeaders: { blobContentType: file.type }, // İçerik tipini (PDF, PNG) belirt
        onProgress: (progressEvent) => {
          // Yükleme yüzdesini hesapla
          const percent = Math.round((progressEvent.loadedBytes / file.size) * 100);
          setProgress(percent);
        }
      });

      // ADIM D: (Opsiyonel) İşlem Başarılı, Veritabanına Kaydet
      // Dosya artık Azure'da. Şimdi kendi DB'mize "Bu dosya şurada duruyor" diye kayıt atabiliriz.
      // await axios.post('/api/files/save-metadata', { fileName: uniqueName, ... });

      setStatus('success');
      alert('Dosya başarıyla yüklendi!');

    } catch (error) {
      console.error("Yükleme hatası:", error);
      setStatus('error');
      alert('Yükleme sırasında bir hata oluştu.');
    }
  };

  return (
    <div style={{ padding: '20px', border: '1px solid #ccc', borderRadius: '8px' }}>
      <h3>Azure Blob Storage Yükleme</h3>
      
      <input type="file" onChange={onFileChange} />
      
      <button 
        onClick={handleUpload} 
        disabled={!file || status === 'loading'}
        style={{ marginLeft: '10px', padding: '5px 15px' }}
      >
        {status === 'loading' ? 'Yükleniyor...' : 'Yükle'}
      </button>

      {/* İlerleme Çubuğu */}
      {progress > 0 && (
        <div style={{ marginTop: '15px' }}>
          <progress value={progress} max="100" style={{ width: '100%' }}></progress>
          <span>%{progress}</span>
        </div>
      )}

      {status === 'success' && <p style={{ color: 'green' }}>✅ İşlem Başarılı!</p>}
      {status === 'error' && <p style={{ color: 'red' }}>❌ Hata Oluştu!</p>}
    </div>
  );
};

export default FileUpload;
```

**Bu Kodda Neler Oluyor? (Kritik Noktalar)**
1. `BlockBlobClient:`
   * Azure SDK'sının sihirli değneğidir. Normalde büyük dosyaları yüklemek için dosyayı 4MB'lık parçalara bölüp, sırayla gönderip, hata alan parçayı tekrar denemeniz gerekir.
   * `uploadData` metodu bunu otomatik yapar. Siz sadece dosyayı verirsiniz, o arka planda en optimize şekilde (paralel yükleme yaparak) Azure'a gönderir.

2. `blobHTTPHeaders:`
   * Bu ayarı yapmazsanız, Azure dosyayı application/octet-stream (bilinmeyen dosya) olarak kaydeder. Tarayıcıda açtığınızda PDF görüntülenmez, direkt indirilir.
   * `blobContentType: file.type` diyerek dosyanın tipini (örn: image/jpeg veya application/pdf) Azure'a bildiriyoruz.

3. Performans:
   * Dosya verisi (Binary Data) asla sizin Backend sunucunuza (localhost:5000) gitmez. Tarayıcıdan direkt Azure sunucularına (blob.core.windows.net) uçar.

## Bölüm 5: Final Entegrasyon ve Veritabanı Kaydı
Dosyayı Azure'a attık ama şu an "kör uçuş" yapıyoruz. Veritabanımızın (SQL Server) bu dosyadan haberi yok. Eğer kullanıcı sayfayı yenilerse dosya kaybolur.

Bu bölümde iki kritik işlemi yapacağız:
1. **Metadata Kaydı:** Yükleme bitince dosya bilgilerini veritabanına işlemek.
2. **Güvenli Görüntüleme:** Dosyaları listelerken, anlık ve süreli "Okuma (Read)" linkleri üretmek.

### 1. Veritabanı Modeli (Backend)
Öncelikle veritabanında neleri saklayacağımıza karar verelim. Dikkat: SAS Token'ı asla veritabanına kaydetmeyin! Tokenlar geçicidir. Biz sadece dosyanın kalıcı adını saklayacağız.

`Models/FileRecord.cs:`
```csharp
public class FileRecord
{
    public int Id { get; set; }
    public string OriginalName { get; set; } // Kullanıcının gördüğü isim (tez.pdf)
    public string StoredName { get; set; }   // Azure'daki benzersiz isim (guid_tez.pdf)
    public string ContentType { get; set; }  // application/pdf
    public long Size { get; set; }           // Byte cinsinden boyut
    public DateTime UploadedAt { get; set; } = DateTime.UtcNow;
    // public string UserId { get; set; }    // Hangi kullanıcı yükledi?
}
```

### 2. API Güncellemesi (Save & List)
`FilesController.cs` dosyamıza iki yeni metot ekliyoruz.

```csharp
[Route("api/[controller]")]
[ApiController]
public class FilesController : ControllerBase
{
    private readonly BlobService _blobService;
    // private readonly AppDbContext _context; // Veritabanı bağlantınız

    public FilesController(BlobService blobService)
    {
        _blobService = blobService;
    }

    // ... Önceki GetUploadUrl metodu burada ...

    // YENİ 1: Yükleme Başarılı Olduğunda Çağrılacak
    [HttpPost("save-metadata")]
    public async Task<IActionResult> SaveFileMetadata([FromBody] FileRecord fileRecord)
    {
        // Gerçek projede: _context.Files.Add(fileRecord); await _context.SaveChangesAsync();
        
        // Simülasyon:
        Console.WriteLine($"Veritabanına kaydedildi: {fileRecord.OriginalName}");
        return Ok(new { message = "Dosya kaydı başarılı" });
    }

    // YENİ 2: Dosyaları Listele ve Okuma Linki Ver
    [HttpGet]
    public IActionResult GetAllFiles()
    {
        // 1. Veritabanından dosyaları çek
        // var files = _context.Files.ToList();
        
        // Simülasyon verisi:
        var files = new List<FileRecord>
        {
            new FileRecord { OriginalName = "fatura.pdf", StoredName = "guid_fatura.pdf" }
        };

        // 2. Her dosya için anlık "Read" token üret
        var fileDtos = files.Select(f => new 
        {
            f.OriginalName,
            // BlobService'e eklediğimiz GenerateSasTokenForRead metodunu kullanıyoruz:
            DownloadUrl = _blobService.GenerateSasTokenForRead(f.StoredName) 
        });

        return Ok(fileDtos);
    }
}
```

### 3. Frontend Güncellemesi (FileUpload.jsx)
React tarafındaki kodumuzun handleUpload kısmına, Azure işlemi bittikten sonra çalışacak olan "Veritabanına Kaydet" adımını ekliyoruz.
```jsx
// ... önceki importlar ve state tanımları

const handleUpload = async () => {
    if (!file) return;

    try {
      setStatus('loading');
      
      // 1. Token Al
      const response = await axios.post('http://localhost:5000/api/files/upload-request', {
        fileName: file.name
      });
      const { sasUrl, uniqueName } = response.data;

      // 2. Azure'a Yükle
      const blobClient = new BlockBlobClient(sasUrl);
      await blobClient.uploadData(file, {
        blobHTTPHeaders: { blobContentType: file.type }
      });

      // 3. (YENİ) Veritabanına Kaydet
      // Burası sadece Azure yüklemesi hatasız biterse çalışır.
      await axios.post('http://localhost:5000/api/files/save-metadata', {
        originalName: file.name,
        storedName: uniqueName, // Azure'daki guid'li isim
        contentType: file.type,
        size: file.size
      });

      setStatus('success');
      alert('Dosya başarıyla yüklendi ve kaydedildi!');
      
      // İsteğe bağlı: Dosya listesini yenileme fonksiyonunu çağır
      // refreshFileList(); 

    } catch (error) {
      // ... hata yönetimi
    }
};
```

### 4. Dosyaları Listeleme ve İndirme (FileList.jsx)
Son olarak, dosyaları kullanıcıya gösteren bileşeni yazalım. Bu bileşen backend'den güvenli, süreli linkleri alacak.

```jsx
import React, { useEffect, useState } from 'react';
import axios from 'axios';

const FileList = () => {
  const [files, setFiles] = useState([]);

  useEffect(() => {
    // Sayfa açılınca dosyaları getir
    axios.get('http://localhost:5000/api/files')
      .then(response => {
        setFiles(response.data);
      })
      .catch(err => console.error(err));
  }, []);

  return (
    <div style={{ marginTop: '20px' }}>
      <h3>Dosyalarım</h3>
      <ul>
        {files.map((file, index) => (
          <li key={index} style={{ marginBottom: '10px' }}>
            {/* Kullanıcı sadece dosya adını görür */}
            <span>{file.OriginalName} </span>
            
            {/* Link tıklandığında Azure'dan iner */}
            <a 
              href={file.DownloadUrl} 
              target="_blank" 
              rel="noopener noreferrer"
              style={{ color: 'blue', textDecoration: 'underline' }}
            >
              [İndir / Görüntüle]
            </a>
          </li>
        ))}
      </ul>
    </div>
  );
};

export default FileList;
```

## Ekstra önlemler
* `Orphaned Files (Yetim Dosyalar):` Nadiren de olsa, dosya Azure'a yüklenir ama kullanıcının interneti kesilir ve veritabanı kaydı yapılamaz. Bunun için haftada bir çalışan bir "Azure vs SQL karşılaştırma" scripti yazılabilir.
* `CDN:` Eğer bu dosyalar tüm dünyaya sunulacaksa, Azure Storage'ın önüne Azure CDN koyarak indirme hızını artırabilirsiniz.