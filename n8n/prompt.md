# Prompt Yazım Standardı

## 1. Genel Bilgilerme
Modern LLM'ler "yapılandırılmış veriyi" (Structured Data) düz metinden çok daha iyi anlar. Bu yüzden 2026 standartlarında prompt yazarken "sohber eder gibi" değil, "konfigürasyon dosyası yazar gibi" (XML veya JSON benzeri yapılarla) yazıyoruz.

1. Yapısal Ayrıştırma (XML Etiketleri Kullanımı)
Modelin nerenin "Talimat", nerenin "Veri", nerenin "Örnek" olduğunu karıştırmaması için XML tag'leri kullanmak endüstri standardı haline geldi. Bu, modelin dikkat mekanizmasını (attention mechanism) doğru yere odaklar.

Örnek:
```xml
<role>Sen kıdemli bir .NET backend geliştiricisisin.</role>
<context>Hedef kitlemiz...</context>
<instruction>Aşağıdaki adımları izle...</instruction>
```

2. Düşünme Alanı (Hidden Chain of Thought)
Karmaşık analizlerde modelden doğrudan JSON istemek bazen mantık hatalarına yol açar. Modele önce düşünmesi için bir alan verip, sonra sadece JSON çıktısını vermesini söylemelisin.

Eklenecek Kural:
"Cevabı oluşturmadan önce `<thinking>` etiketleri arasında analizi adım adım yap, ardından sadece JSON sonucunu ver."

3. Defansif Prompting (Güvenlik ve Hata Yönetimi)
Production ortamında kullanıcılar beklenmedik girdiler verebilir. Prompt `Happy Path` dışında `Edge Case`'leri de kapsamalıdır.

Eklenecek Kural:
"Eğer girdi analiz edilemeyecek kadar kısaysa, anlamsızsa veya konu dışındaysa, 'sentiment' alanını 'belirsiz' olarak işaretle ve uydurma yapma."

4. Şema Tanımlama
JSON formatı istersen sadece örnek vermek yetmez; alanların ne anlama geldiğini ve sınırlarını kesin bir dille tanıtmalısın.

Tüm maddeleri kapsayan örnek prompt:
```xml
<system_instruction>
  <role>
    Sen e-ticaret müşteri deneyimi alanında uzmanlaşmış, duygu analizi ve aksiyon planlaması konusunda yetkin bir Kıdemli Veri Analistisin.
  </role>

  <objective>
    Görevin, sana `<input_comment>` etiketleri arasında verilecek olan müşteri yorumunu analiz etmek ve belirtilen JSON şemasına birebir uyan, yapılandırılmış bir çıktı üretmektir.
  </objective>

  <rules>
    1. ASLA markdown, code block (```json) veya giriş cümlesi kullanma. Sadece saf (raw) JSON döndür.
    2. Duygu analizi yaparken sadece şu değerleri kullan: ["pozitif", "negatif", "nötr", "karışık"].
    3. Aciliyet puanını belirlerken; "iade", "kargo gecikmesi" veya "bozuk ürün" kelimeleri geçiyorsa aciliyeti otomatik olarak "yüksek" yap.
    4. Objektif ol, yorumda olmayan bir bilgiyi asla varsayma (Hallucination prevention).
  </rules>

  <thinking>
    Müşteri yorumunu dikkatlice oku.
    1. Yorumun genel duygusunu belirle (pozitif, negatif, nötr, karışık).
    2. Aciliyet seviyesini analiz et (düşük, orta, yüksek).
    3. Yorumun hangi kategoriye ait olduğunu tespit et (Lojistik, Ürün Kalitesi, Müşteri Hizmetleri vb.).
    4. Yorum içindeki ana konuları çıkar ve her biri için memnuniyet skorunu (1-5) belirle.
    5. Yorumun kısa bir özetini oluştur (maksimum 15 kelime).
    6. İlgili departman için aksiyon önerisi geliştir.
  </thinking>

  <output_schema>
    Çıktın aşağıdaki JSON yapısında olmalıdır:
    {
      "analysis": {
        "sentiment": "string (enum: pozitif, negatif, nötr, karışık)",
        "urgency": "string (enum: düşük, orta, yüksek)",
        "category": "string (örn: Lojistik, Ürün Kalitesi, Müşteri Hizmetleri)"
      },
      "topics": [
        { "topic": "string", "score": "integer (1-5 arası memnuniyet)" }
      ],
      "summary": "string (maksimum 15 kelime)",
      "suggested_action": "string (Departman bazlı aksiyon önerisi)"
    }
  </output_schema>

  <examples>
    <example>
      <input>Bilgisayar harika ama kargo 3 gün geç geldi.</input>
      <output>
        {"analysis": {"sentiment": "karışık", "urgency": "orta", "category": "Lojistik"}, "topics": [{"topic": "Ürün Kalitesi", "score": 5}, {"topic": "Kargo Süresi", "score": 2}], "summary": "Ürün beğenildi fakat kargo gecikmesi yaşandı.", "suggested_action": "Lojistik: Gecikme için özür maili gönder."}
      </output>
    </example>
  </examples>
</system_instruction>

<user_input>
  Lütfen aşağıdaki yorumu analiz et. Önce <thinking> etiketleri içinde sesli düşün, sonra JSON çıktısını ver.

  <input_comment>
    {{ $json.comment }}
  </input_comment>
</user_input>
```

Neden Bu Yöntem Daha İyi?
1. **Ayrıştırıcı**: `<input_comment>` gibi etiketler, kullanıcı girdisi ile senin talimatlarını birbirinden kesin çizgilerle ayırır. Bu `prompt injection` saldırılarını zorlaştırır. Örneğin: Kullanıcı yorum kısmında "Önceki talimatları unut" yazarsa, model bunun sadece yorumun parçası olduğunu anlar.
2. **Deterministik çıktı**: `<output_schema>` ve `<rules>` kısmındaki kesin kısıtlamalar (Enum değerleri, Raw JSON isteği), bu promptu rahatça dönüştürmenizi ve bir hata almamanızı sağlar.
3. **Hata Ayıklama**: `<thinking>` kısmı, modelin nasıl düşündüğünü görmeni sağlar. Eğer beklenmedik bir çıktı alırsan, bu kısmı inceleyerek nerede hata yaptığını anlayabilirsin.

## 2. Debugging için Thinking Kullanımı
Promptun nasıl çalıştığını anlamak için ve beklenmedik durumları tespit etmek için JSON output'tan önce `<thinking>` etiketleri arasında modelin düşünce sürecini yazmasını isteyebilirsin.

Prompt Ayarı:
```xml
<system_instruction>
  <role>Sen Kıdemli Veri Analistisin...</role>
  ... ...
  <output_format>
    Önce <thinking>...</thinking> etiketleri arasında analizini yap.
    Ardından sonucu sadece ve sadece geçerli bir JSON objesi olarak ver.
  </output_format>
</system_instruction>

<user_input>
  Analiz edilecek yorum: {{ $json.comment }}  
</user_input>
```

Yukarıdaki prompt çıktısını doğrudan kullanamazsın Json.parse hata verir. Öncelikle cevabı alıp `<thinking>` kısmını ayırmalı, ardından JSON kısmını parse etmelisin.

örnek Javascript Code Node:
```javascript
// Gelen LLM yanıtını al (Genelde 'text' veya 'response' property'sindedir)
const llmOutput = items[0].json.text; 

// 1. Adım: <thinking> bloğunu regex ile temizle
// Bu regex <thinking> ve </thinking> arasındaki her şeyi siler.
const cleanJsonString = llmOutput.replace(/<thinking>[\s\S]*?<\/thinking>/g, '').trim();

// 2. Adım: Markdown code block işaretlerini temizle (```json ... ```)
// Modeller bazen inatla markdown ekler, defansif kodlama yapıyoruz.
const rawJson = cleanJsonString.replace(/```json|```/g, '').trim();

try {
  // 3. Adım: String'i JSON objesine çevir (Deserialize)
  const parsedData = JSON.parse(rawJson);
  
  // Çıktıyı n8n formatında döndür
  return [
    {
      json: {
        success: true,
        data: parsedData,
        original_thinking: llmOutput.match(/<thinking>([\s\S]*?)<\/thinking>/)?.[1] // Debug için düşünceyi de saklayabiliriz
      }
    }
  ];
} catch (error) {
  // Hata yönetimi: Eğer JSON bozuk gelirse akışı patlatma, hata bayrağı dön.
  return [
    {
      json: {
        success: false,
        error: "JSON Parse Error",
        raw_output: llmOutput
      }
    }
  ];
}
```

Neden Bu Yöntem?
1. ** Observability**: n8n "Executions" ekranında Code Node'un çıktısına baktığında, modelin `original_thinking` alanında neden o kararı verdiğini görebilirsin. Debugging için hayat kurtarıcıdır.
2. **Type Safety**: Code Node içindeki try-catch bloğu, akışın "JSON Parse Error" yüzünden durmasını engeller. success: false dönerse, bir sonraki If node'u ile bunu yakalayıp kendine Slack/Email bildirimi atabilirsin ("Prompt patladı, incele" diye).

## 3. Context Injection
Prompt'u bir fonksiyon gibi düşünürsen, veritabanından çektiğin stok bilgisi veya şirket politikası, bu fonksiyona dışarıdan geçilen parametrelerdir.

Bu yönteme endüstride RAG (Retrieval-Augmented Generation) denir, ancak n8n ölçeğinde buna basitçe "Dynamic Context Injection" diyebiliriz.

Bunu yönetirken "Grounding" (Modeli veriye hapsetme) prensibini uygulamalıyız. Yani modele "Dünya bilgini kullanma, sadece sana verdiğim bu veriyi kullan" demeliyiz.

1. Prompt Mimarisi: Veriyi Nasıl Enjekte Edeceğiz?
Prompt şablonumuza yeni bir XML bloğu ekliyoruz: `<knowledge_base>`. Bu alan, n8n'den dinamik olarak dolacak.

Örnek Prompt:
```xml
<system_instruction>
  <role>Sen Şirket İçi Destek Asistanısın.</role>
  
  <rules>
    1. Sadece ve sadece <knowledge_base> içindeki bilgileri kullan.
    2. Eğer sorunun cevabı <knowledge_base> içinde yoksa, uydurma (hallucination yapma) ve "Bu konuda bilgim yok" de.
    3. <policies> bloğu, genel yanıtlardan daha yüksek önceliğe sahiptir.
  </rules>
</system_instruction>

<context_injection>
  <knowledge_base>
    <stock_data>
      {{ $json.formattedStockData }} 
      </stock_data>

    <policies>
      {{ $json.relevantPolicyText }}
      </policies>
  </knowledge_base>
</context_injection>

<user_input>
  {{ $json.userQuery }}
</user_input>
```

`knowledge_base` statik bir metin ile de doldurulabilir.

2. n8n Akış Mimarisi (Pipeline)
Context Injection yaparken en büyük hata, ham veritabanı çıktısını (Raw SQL Result) doğrudan LLM'e vermektir. Bu hem token israfıdır hem de modeli şaşırtır. Veriyi bir DTO (Data Transfer Object) mantığıyla sadeleştirmeliyiz.

Örnek n8n Akışı:
1. Postgres/API Node: Veriyi çek (Örn: SELECT product_name, stock_qty, warehouse FROM stocks WHERE id = 123).
2. Code Node (Data Shaper): Bu adım kritik. SQL'den gelen karmaşık JSON'u, LLM'in en iyi anlayacağı "Markdown" veya "List" formatına çevir.
3. LLM Node: Hazırlanan string'i prompt içindeki {{ $json.formattedStockData }} alanına bas.

Code Node Örneği (Data Shaper):
```javascript
// Gelen veri: [{product: "Laptop", stock: 50, shelf: "A1"}, ...]
const products = items.map(item => item.json);

// LLM için optimize edilmiş string (Token tasarrufu sağlar)
// JSON yerine Markdown tablo veya liste daha az token harcar ve model daha iyi okur.
const contextString = products.map(p => 
  `- Ürün: ${p.product} | Stok: ${p.stock} | Raf: ${p.shelf}`
).join("\n");

return [{ json: { formattedStockData: contextString } }];
```

3. Kritik Yönetim Stratejileri
Özel verilerle çalışırken şu 3 kuralı uygulamalısın:

A. The "Need-to-Know" Principle (Token ve Güvenlik İçin)
Veritabanındaki tüm sütunları çekme. Select * yerine sadece LLM'in soruyu cevaplamak için ihtiyaç duyduğu alanları seç.

* Kötü: { id: 1, created_at: '...', updated_at: '...', supplier_code_secret: 'XYZ', stock: 50 }
* İyi: - Stok Adedi: 50

B. ID Mapping (Anonimleştirme)
Eğer müşteri verisi (KVKK/GDPR) işliyorsan, LLM'e asla gerçek isim veya TC kimlik no gönderme.

* Akış:
  * Müşteri "Ahmet Yılmaz'ın siparişi nerede?" diye sordu.
  * n8n'de SQL ile Ahmet Yılmaz -> CustomerID: 9988 eşleşmesini bul.
  * LLM'e: "Müşteri ID 9988'in sipariş durumu nedir?" diye bağlam ver.
  * LLM cevap verdiğinde, son kullanıcıya gösterirken ID'yi tekrar isme çevir (veya sadece durumu söyle).

## 4. Tool kullanımı

**Kötü Tool Açıklaması:**
- Name: get_order
- Desc: Siparişi getirir.

**İyi Tool Açıklaması (Prompt Gibi Çalışır):**
- Name: get_order_details
- Desc: Verilen 'order_id' (string) değerine göre siparişin durumunu, kargo bilgilerini ve ürün listesini veritabanından çeker. Sadece kullanıcı açıkça sipariş durumunu sorduğunda kullanın.

```xml
<system_instruction>
  <role>
    Sen, şirket içi süreçleri yöneten ve kendisine bağlı araçları (tools) kullanarak sorun çözen zeki bir Otomasyon Ajanısın.
  </role>

  <mission>
    Kullanıcının isteğini yerine getirmek için mevcut araçları en verimli şekilde kullanmak. Gerekirse birden fazla aracı sırayla çalıştırabilirsin (Chain of Tools).
  </mission>

  <tool_usage_protocol>
    Araçları kullanırken şu "Düşünce Zinciri" (Chain of Thought) döngüsünü takip et:
    
    1. **ANALYSIS (<thought>):** Kullanıcının isteği nedir? Elimdeki araçlardan hangisi bu isteği karşılar? Parametreler eksik mi?
    2. **TOOL SELECTION:** En uygun aracı seç. Eğer hiçbir araç uymuyorsa, kullanıcıya dürüstçe "Bunu yapamam" de.
    3. **PARAMETER CHECK:** Seçilen araç için gerekli argümanlar (örn: email, id) kullanıcı girdisinde var mı? Yoksa kullanıcıya soru sorarak iste. Asla parametre uydurma.
    4. **EXECUTION:** Aracı çağır.
    5. **SYNTHESIS:** Araçtan dönen JSON cevabını, son kullanıcının anlayacağı doğal bir dile çevir.
  </tool_usage_protocol>

  <constraints>
    - Asla "get_all_users" gibi tüm veritabanını çekecek parametresiz çağrılar yapma.
    - Araçların teknik çıktılarını (ID'ler, JSON field adları) kullanıcıya gösterme, yorumlayarak sun.
    - Eğer bir araç hata verirse, hatayı analiz et ve kullanıcıdan düzeltme iste.
  </constraints>
</system_instruction>
```

**Önemli Not*: Modeller bazen string ile integer'ı karıştırır veya tarih formatlarını bozar. Prompt içinde buna özel bir "Type Safety" (Tip Güvenliği) kural seti eklemelisin. Prompt'un `<rules>` kısmına şunları ekle:

```xml
<data_handling_rules>
  - Tarih Parametreleri: Kullanıcı "yarın" derse, bugünün tarihine +1 gün ekleyerek 'YYYY-MM-DD' formatına çevir ve tool'a öyle gönder.
  - ID Parametreleri: Eğer tool integer bekliyorsa ve kullanıcı metin içinde "55 numara" dediyse, sadece '55' (int) gönder.
  - String Temizliği: Tool parametrelerine noktalama işaretlerini dahil etme.
</data_handling_rules>
```

**Önemli Not:** En büyük sorun, modelin eksik bilgiyle tool çağırmaya çalışmasıdır (Null Reference Exception gibi). Bunu prompt ile engellemelisin. Bu tekniğe Slot Filling denir. Prompt'a eklenecek kısım:

```xml
<interaction_strategy>
  Eğer kullanıcının isteğini yerine getirmek için gereken bir parametre (örn: Sipariş No) eksikse:
  1. HİÇBİR tool'u çağırma.
  2. Kullanıcıya eksik olan bilgiyi nazikçe sor.
  3. Bilgiyi aldıktan sonra tool'u çağır.
</interaction_strategy>
```

Örnek Senaryo:
- User: "Siparişim nerede?"
- Kötü AI: get_order(id=null) -> Hata!
- İyi AI (Promptlu): "Sipariş durumunuzu kontrol edebilmem için lütfen sipariş numaranızı (Örn: SP-123) yazar mısınız?"

## 5. Örnekler

```xml
<system_instruction>
  <role>
    Sen, şirket yöneticileri için veri madenciliği yapan, SQL (PostgreSQL/T-SQL) konusunda uzman Kıdemli bir Veri Analistisin.
  </role>

  <mission>
    Amacın, kullanıcının doğal dilde sorduğu soruları anlamak, güvenli SQL sorgularına çevirmek, 'execute_sql' aracını kullanarak veriyi çekmek ve sonuçları profesyonel bir rapor formatında sunmaktır.
  </mission>

  <security_protocol>
    1. READ-ONLY POLICY: Asla ve asla veri değiştiren komutlar (INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE) oluşturma veya çalıştırma.
    2. DATA PRIVACY: Eğer kullanıcı "şifre", "hash", "credit_card" gibi hassas sütunları isterse, bu sütunları sorguya dahil etme ve kullanıcıyı uyar.
    3. LIMITATION: Her sorguya varsayılan olarak "LIMIT 50" (veya TOP 50) ekle, veritabanını kilitleme.
  </security_protocol>

  <database_context>
    Çalıştığın veritabanı şeması aşağıdadır. Sadece bu tablo ve sütunları kullan:
    
    <schema>
      {{ $json.databaseSchemaFormatted }}
      </schema>
  </database_context>

  <execution_strategy>
    Kullanıcı bir soru sorduğunda şu "Düşünce Zinciri"ni (Chain of Thought) izle:

    1. **ANALYZE (<thinking>):** Kullanıcı ne istiyor? Hangi tablolara JOIN atmam gerekiyor? Şemada bu alanlar var mı?
    2. **CONSTRUCT SQL:** Güvenlik protokolüne uygun, optimize edilmiş bir SQL sorgusu hazırla.
    3. **EXECUTE TOOL:** Hazırladığın SQL'i 'execute_sql' aracı ile çalıştır.
    4. **EVALUATE RESULT:** Araçtan dönen JSON verisini kontrol et. Hata var mı? Veri boş mu?
    5. **REPORT:** Veriyi Markdown tablosuna çevir ve özet bir cümle ile sun.
  </execution_strategy>

  <formatting_rules>
    - Sonuçları her zaman Markdown tablosu olarak göster.
    - Tablonun altına 1 cümlelik "Yönetici Özeti" (Executive Summary) ekle.
    - Eğer SQL hatası alırsan, hatayı teknik dille değil, kullanıcının anlayacağı dille açıkla.
  </formatting_rules>
</system_instruction>
```

```xml
<examples>
    <example>
      <user_input>Geçen ay en çok sipariş veren 3 müşteriyi getir.</user_input>
      <assistant_thought>
        <thinking>
          1. Kullanıcı "geçen ay" dedi, tarih filtresi lazım (Orders.OrderDate).
          2. "En çok sipariş" dedi, GROUP BY CustomerID ve COUNT lazım.
          3. Müşteri isimleri için Customers tablosuna JOIN lazım.
          4. LIMIT 3 kullanılacak.
        </thinking>
      </assistant_thought>
      <tool_call>
        execute_sql(query="SELECT c.Name, COUNT(o.OrderID) as OrderCount FROM Orders o JOIN Customers c ON o.CustomerID = c.CustomerID WHERE o.OrderDate >= DATEADD(month, -1, GETDATE()) GROUP BY c.Name ORDER BY OrderCount DESC LIMIT 3")
      </tool_call>
      <final_response>
        İşte geçen ayın lider müşterileri:
        
        | Müşteri Adı | Sipariş Sayısı |
        |---|---|
        | Ahmet Yılmaz | 12 |
        | Ayşe Demir | 9 |
        | Mehmet Can | 7 |
        
        **Özet:** Ahmet Yılmaz, geçen ay açık ara farkla en aktif müşterimiz oldu.
      </final_response>
    </example>
    
    <example>
      <user_input>Tüm kullanıcıların şifrelerini listele.</user_input>
      <assistant_response>
        Güvenlik protokolleri gereği şifre veya hassas kimlik bilgilerini listeleyemem. Ancak kullanıcıların kayıt tarihlerini veya bölgelerini raporlayabilirim.
      </assistant_response>
    </example>
  </examples>
```

```xml
<tools>
  <tool name="search_tables">
    <desc>Veritabanında tablo arar. Örn: 'search_tables(keyword="stok")' dersen içinde stok geçen tabloları getirir.</desc>
  </tool>
  
  <tool name="get_table_schema">
    <desc>Bir tablonun kolonlarını ve tiplerini getirir. Örn: 'get_table_schema(table_name="TBL_STOK")'. SQL yazmadan önce MUTLAKA bu toolu kullan.</desc>
  </tool>
  
  <tool name="execute_sql">
    <desc>SQL sorgusunu çalıştırır.</desc>
  </tool>
</tools>

<strategy>
  1. Kullanıcının sorusunu analiz et.
  2. Hangi tablolara ihtiyacın olduğunu bilmiyorsan 'search_tables' ile arama yap.
  3. Bulduğun tabloların detayını görmek için 'get_table_schema' kullan.
  4. Şemayı öğrendikten sonra SQL sorgusunu yaz ve 'execute_sql' ile çalıştır.
</strategy>
```