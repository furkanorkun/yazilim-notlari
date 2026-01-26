# TanStack Query: staleTime vs gcTime 🕒

TanStack Query (eski adıyla React Query) kullanırken bu iki kavram en çok karıştırılan konulardan biridir, o yüzden kafanızın karışması çok normal. En basit haliyle fark şudur:

- **staleTime**: Verinin ne zaman yeniden isteneceğini (refetch) kontrol eder.
- **gcTime**: Verinin hafızadan (RAM) ne zaman silineceğini kontrol eder.

Aşağıda bu farkı netleştirecek detayları, bir senaryoyu ve görselleştirmeyi bulabilirsiniz.

## 1. staleTime (Bayatlama Süresi) ⏳

**Bu veri ne kadar süreyle 'taze' kabul edilir?**

- **Varsayılan Değer**: 0 (Veri gelir gelmez bayat kabul edilir).
- **Ne İşe Yarar**: Eğer veri "taze" (fresh) ise, TanStack Query sunucuya tekrar istek atmaz; önbellekteki veriyi kullanır. Süre dolduğunda veri "bayat" (stale) olur.
- **Bayat Olursa Ne Olur?**: Veri hala ekranda görünür, kaybolmaz. Ancak kullanıcı sayfaya tekrar odaklandığında veya bileşen tekrar render olduğunda, TanStack Query arka planda sessizce yeni veriyi sunucudan çeker ve ekranı günceller.
- **Özet**: staleTime ağ trafiğini (network requests) azaltmak içindir.

## 2. gcTime (Garbage Collection Time) 🗑️

**Kullanılmayan veri hafızada ne kadar tutulsun?**

- **Eski Adı**: cacheTime (v5 öncesi).
- **Varsayılan Değer**: 5 dakika.
- **Ne İşe Yarar**: Bir veriyi kullanan hiçbir bileşen (component) ekranda kalmadığında (unmount olduğunda), veri "inactive" (pasif) duruma geçer. Bu anda geri sayım başlar.
- **Süre Dolarsa Ne Olur?**: Süre dolana kadar kullanıcı o sayfaya geri dönerse, veri hemen gösterilir. Ancak süre dolarsa, veri hafızadan tamamen silinir. Kullanıcı sayfaya geri dönerse, "loading" (yükleniyor) durumu tekrar yaşanır çünkü veri sıfırdan çekilmelidir.
- **Özet**: gcTime hafıza yönetimini ve verinin kalıcılığını sağlar.

## Somut Bir Senaryo Üzerinden Bakalım 📋

Diyelim ki bir "Profil Sayfası"nız var. Ayarlarınız şöyle:

- **staleTime**: 10 saniye
- **gcTime**: 5 dakika

### Adım Adım Olay Akışı:

1. **Kullanıcı Profil sayfasına girdi**: Veri sunucudan çekilir. Ekranda gösterilir.
2. **İlk 10 saniye içinde**: Kullanıcı başka bir sekmeye gidip hemen geri dönerse, veri Taze (Fresh) olduğu için hiçbir ağ isteği (request) atılmaz. Önbellekten anında gösterilir.
3. **11. saniyede (Veri artık Bayat)**: Kullanıcı sayfada durmaya devam ediyor. Veri ekranda durur, bir değişiklik olmaz. Ancak kullanıcı sayfayı yenilerse veya başka bir tab'e gidip gelirse, TanStack Query "Veri bayatlamış, arka planda güncelleyeyim" der. Kullanıcı eski veriyi görmeye devam ederken, arka planda yeni veri çekilir ve gelince ekran güncellenir.
4. **Kullanıcı "Ayarlar" sayfasına gitti (Profil component'i unmount oldu)**: Profil verisi artık kullanılmıyor ("Inactive" oldu). gcTime sayacı (5 dakika) şimdi çalışmaya başlar.
5. **2 dakika sonra kullanıcı Profil sayfasına geri döndü**: gcTime henüz dolmadığı için veri hafızadadır. Veri anında ekrana gelir (Loading spinner dönmez). Ancak veri "bayat" olduğu için arka planda güncelleme yapılır.
6. **Eğer kullanıcı 10 dakika sonra dönseydi**: gcTime dolduğu için veri silinmiş olacaktı. Kullanıcı sayfaya girdiğinde Loading spinner (yükleniyor) görecekti ve veri sıfırdan çekilecekti.

## Karşılaştırma Tablosu 📊

| Özellik              | staleTime                          | gcTime                              |
|----------------------|------------------------------------|-------------------------------------|
| **Neyi Kontrol Eder?** | Yeniden isteme (Refetch) sıklığını. | Verinin hafızada kalma süresini.    |
| **Süre Ne Zaman İşler?** | Veri fetch edildiği andan itibaren. | Component ekrandan gidince (inactive olunca). |
| **Varsayılan Değer** | 0 (Hemen bayatlar).               | 5 dakika.                          |
| **Süre Dolunca Ne Olur?** | Sonraki tetiklenmede arka planda fetch yapılır. | Veri çöp kutusuna atılır (silinir). |
| **Kullanıcıya Etkisi** | Verinin güncelliğini belirler.     | "Loading" ekranı görüp görmeyeceğini belirler. |