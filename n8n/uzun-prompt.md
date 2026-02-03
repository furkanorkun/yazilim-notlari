# Token Limit
Modellerin context pencereleri çok genişlese de, gereksiz veri göndermek hem maliyeti artırır hem de Latency (gecikme) yaratır. Ayrıca çok fazla gürültü (noise), modelin odağını dağıtır ("Lost in the Middle" fenomeni).

1. Strateji: "Compressor" Middleware (Özetleyerek Küçültme)
Kullanıcıdan gelen metin çok uzunsa (örneğin 5.000 kelimelik bir makale yapıştırdı ve "bunu düzelt" dedi), bunu doğrudan ana pahalı modele (Main Agent) göndermek yerine, araya ucuz ve hızlı bir model koyarız.

Buna Map-Reduce deseni diyebiliriz.
* Adım 1 (Router): Gelen metnin uzunluğunu ölç (n8n len() fonksiyonu).
* Adım 2 (Compressor Node): Eğer metin > 2000 karakter ise, Gemini Flash veya GPT-4o-mini gibi ucuz bir modele gönder.
  * Prompt: "Aşağıdaki metni, ana fikri ve kritik detayları kaybetmeden %30 oranında özetle ve yapılandır." 
* Adım 3 (Main Agent): Sıkıştırılmış (distilled) veriyi asıl işi yapacak olan modele gönder.

2. Strateji: RAG ile Dinamik Chunking (Akıllı Seçim)
Eğer kullanıcı uzun bir metin gönderiyor ve bunun tamamının işlenmesi gerekmiyorsa (örneğin: "Bu sözleşmedeki cezai şartlar nelerdir?" diye sorup 50 sayfa PDF attıysa), Chunking burada devreye girer.

Sıralı (Sequential) Chunking yerine Semantik Chunking kullanmalısın.
1. Ingestion: Metni n8n içinde parçalara böl (örn: 500 tokenlık parçalar).
2. Vector Store: Bu parçaları geçici bir Vector Store'a (Pinecone veya n8n'in kendi in-memory vektör desteği) göm (embed et).
3. Retrieval: Kullanıcının sorusunu ("cezai şartlar") vektör veritabanında arat.
4. Generation: Sadece en alakalı 3-4 parçayı (chunk) alıp ana prompt'a ekle.

3. Strateji: Recursive Summarization (Zincirleme Özet)
Eğer metin, Context Window'a sığmayacak kadar büyükse (örneğin bir kitap), Chunking tek başına yetmez.

1. Metni Chunk'lara böl (A, B, C, D).
2. A'yı özetle -> Özet_A.
3. Özet_A + B'yi modele ver ve "Önceki özeti dikkate alarak B'yi de kapsayan yeni bir özet çıkar" de -> Özet_AB.
4. Bu şekilde sona kadar git. Sonuçta elinde tüm metnin "yoğunlaştırılmış" hali kalır.