# Events (Olaylar)
Event, bir şey olduğunda diğer nesneleri haberdar etmek için kullanılan bir mekanizmadır. Delegate'ler üzerine inşa edilmiştir: bir delegate birden fazla metodu tutabilir, event ise bu delegate'in dışarıdan yalnızca += ve -= ile abone olunabilen, ama doğrudan çağrılamayan özel bir versiyonudur.

Tipik akış: **Yayıncı (publisher)** bir event tanımlar → **Abone (subscriber)** o event'e bir metot bağlar → Yayıncı event'i tetiklediğinde abone metodları otomatik çağrılır.

## Temel Tanım

```csharp
// 1. Delegate tipini tanımla
public delegate void StockAlertHandler(string stockName, decimal price);

public class StockMarket
{
    // 2. event anahtar kelimesiyle event'i tanımla
    public event StockAlertHandler? PriceChanged;

    private decimal _price;

    public void UpdatePrice(string stockName, decimal newPrice)
    {
        _price = newPrice;
        // 3. Event'i tetikle (null kontrolü zorunlu — hiç abone yoksa null olur)
        PriceChanged?.Invoke(stockName, newPrice);
    }
}

// 4. Abone ol ve dinle
StockMarket market = new StockMarket();
market.PriceChanged += (name, price) => Console.WriteLine($"{name} fiyatı: {price:C}");

market.UpdatePrice("THYAO", 234.50m); // Çıktı: THYAO fiyatı: ₺234,50
```

## EventHandler ve EventHandler<TEventArgs>
Her event için ayrı delegate tanımlamak yerine, .NET'in hazır delegate türlerini kullanmak standart yaklaşımdır.

- `EventHandler` → ek veri taşımayan eventler için
- `EventHandler<TEventArgs>` → özel veri taşıyan eventler için

```csharp
public class Door
{
    // EventHandler: (object sender, EventArgs e) imzasını kullanır
    public event EventHandler? Opened;
    public event EventHandler? Closed;

    public void Open()
    {
        Console.WriteLine("Kapı açılıyor...");
        // sender olarak 'this' geçirilir — abone hangi nesnenin event'i fırlattığını bilir
        Opened?.Invoke(this, EventArgs.Empty);
    }

    public void Close()
    {
        Console.WriteLine("Kapı kapanıyor...");
        Closed?.Invoke(this, EventArgs.Empty);
    }
}

Door frontDoor = new Door();
frontDoor.Opened += (sender, e) => Console.WriteLine("Alarm: Ön kapı açıldı!");
frontDoor.Closed += (sender, e) => Console.WriteLine("Alarm: Ön kapı kapandı.");

frontDoor.Open();
// Kapı açılıyor...
// Alarm: Ön kapı açıldı!
```

## Custom EventArgs
Event ile birlikte veri taşımak için `EventArgs`'tan türetilmiş bir sınıf oluşturulur.

```csharp
// Sipariş verildiğinde taşınacak veriler
public class OrderPlacedEventArgs : EventArgs
{
    public int OrderId { get; }
    public string CustomerName { get; }
    public decimal TotalAmount { get; }

    public OrderPlacedEventArgs(int orderId, string customerName, decimal totalAmount)
    {
        OrderId = orderId;
        CustomerName = customerName;
        TotalAmount = totalAmount;
    }
}

public class OrderService
{
    // EventHandler<T>: (object sender, OrderPlacedEventArgs e) imzasını kullanır
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    public void PlaceOrder(int orderId, string customerName, decimal total)
    {
        Console.WriteLine($"Sipariş #{orderId} oluşturuldu.");

        var args = new OrderPlacedEventArgs(orderId, customerName, total);
        OrderPlaced?.Invoke(this, args);
    }
}
```

### Kullanımı
```csharp
OrderService orderService = new OrderService();

// E-posta servisi abone olur
orderService.OrderPlaced += (sender, e) =>
    Console.WriteLine($"[Email] {e.CustomerName}'e sipariş onayı gönderildi. Tutar: {e.TotalAmount:C}");

// Kargo servisi abone olur
orderService.OrderPlaced += (sender, e) =>
    Console.WriteLine($"[Kargo] Sipariş #{e.OrderId} kargo sistemine iletildi.");

// Muhasebe servisi abone olur
orderService.OrderPlaced += (sender, e) =>
    Console.WriteLine($"[Muhasebe] {e.TotalAmount:C} tutarındaki sipariş faturası oluşturuluyor.");

orderService.PlaceOrder(1001, "Ahmet Yılmaz", 549.90m);
// Sipariş #1001 oluşturuldu.
// [Email] Ahmet Yılmaz'e sipariş onayı gönderildi. Tutar: ₺549,90
// [Kargo] Sipariş #1001 kargo sistemine iletildi.
// [Muhasebe] ₺549,90 tutarındaki sipariş faturası oluşturuluyor.
```

## Abonelikten Çıkma (-=)
+= ile abone olunan metot, -= ile abonelikten çıkarılabilir. Özellikle nesne ömrü yönetiminde önemlidir — çıkılmazsa bellek sızıntısı olabilir.

```csharp
public class TemperatureSensor
{
    public event EventHandler<double>? TemperatureChanged;

    public void SetTemperature(double temp)
    {
        TemperatureChanged?.Invoke(this, temp);
    }
}

void OnTemperatureChanged(object? sender, double temp)
    => Console.WriteLine($"Sıcaklık: {temp}°C");

TemperatureSensor sensor = new TemperatureSensor();
sensor.TemperatureChanged += OnTemperatureChanged;

sensor.SetTemperature(22.5); // Sıcaklık: 22,5°C
sensor.SetTemperature(23.1); // Sıcaklık: 23,1°C

// Abonelikten çık
sensor.TemperatureChanged -= OnTemperatureChanged;

sensor.SetTemperature(24.0); // Çıktı yok — artık dinlenmiyor
```

## Gerçek Dünya Örneği: Stok Yönetim Sistemi

Bir depoda ürün stoku azaldığında birden fazla birimin (bildirim, satın alma, raporlama) haberdar edilmesi gerekir.

```csharp
public class StockChangedEventArgs : EventArgs
{
    public string ProductName { get; }
    public int CurrentStock { get; }
    public int Threshold { get; }

    public StockChangedEventArgs(string productName, int currentStock, int threshold)
    {
        ProductName = productName;
        CurrentStock = currentStock;
        Threshold = threshold;
    }
}

public class InventoryService
{
    public event EventHandler<StockChangedEventArgs>? LowStockDetected;

    private readonly Dictionary<string, int> _stocks = new();

    public void AddProduct(string name, int quantity) => _stocks[name] = quantity;

    public void Sell(string productName, int quantity)
    {
        if (!_stocks.ContainsKey(productName)) return;

        _stocks[productName] -= quantity;
        int current = _stocks[productName];

        Console.WriteLine($"{productName} satışı: {quantity} adet. Kalan: {current}");

        const int threshold = 10;
        if (current <= threshold)
        {
            var args = new StockChangedEventArgs(productName, current, threshold);
            LowStockDetected?.Invoke(this, args);
        }
    }
}

// Bildirim servisi
public class NotificationService
{
    public void OnLowStock(object? sender, StockChangedEventArgs e)
        => Console.WriteLine($"[Bildirim] UYARI: '{e.ProductName}' stoğu kritik seviyede! Kalan: {e.CurrentStock}");
}

// Satın alma servisi
public class PurchasingService
{
    public void OnLowStock(object? sender, StockChangedEventArgs e)
        => Console.WriteLine($"[Satın Alma] '{e.ProductName}' için otomatik sipariş talebi oluşturuldu.");
}
```

### Kullanımı
```csharp
InventoryService inventory = new InventoryService();
NotificationService notifications = new NotificationService();
PurchasingService purchasing = new PurchasingService();

// Abonelikler
inventory.LowStockDetected += notifications.OnLowStock;
inventory.LowStockDetected += purchasing.OnLowStock;

inventory.AddProduct("Laptop", 25);

inventory.Sell("Laptop", 10); // Laptop satışı: 10 adet. Kalan: 15
inventory.Sell("Laptop", 8);
// Laptop satışı: 8 adet. Kalan: 7
// [Bildirim] UYARI: 'Laptop' stoğu kritik seviyede! Kalan: 7
// [Satın Alma] 'Laptop' için otomatik sipariş talebi oluşturuldu.
```

## Gerçek Dünya Örneği: Kullanıcı Oturum Yönetimi

Kullanıcı giriş/çıkış yaptığında farklı servislerin (loglama, güvenlik, UI) tepki vermesi.

```csharp
public class UserSessionEventArgs : EventArgs
{
    public string Username { get; }
    public DateTime Timestamp { get; }
    public string IpAddress { get; }

    public UserSessionEventArgs(string username, string ipAddress)
    {
        Username = username;
        IpAddress = ipAddress;
        Timestamp = DateTime.Now;
    }
}

public class AuthService
{
    public event EventHandler<UserSessionEventArgs>? UserLoggedIn;
    public event EventHandler<UserSessionEventArgs>? UserLoggedOut;

    public bool Login(string username, string password, string ip)
    {
        // Gerçek doğrulama mantığı burada olurdu
        bool isValid = password == "sifre123";

        if (isValid)
        {
            Console.WriteLine($"{username} girişi başarılı.");
            UserLoggedIn?.Invoke(this, new UserSessionEventArgs(username, ip));
        }

        return isValid;
    }

    public void Logout(string username, string ip)
    {
        Console.WriteLine($"{username} çıkış yaptı.");
        UserLoggedOut?.Invoke(this, new UserSessionEventArgs(username, ip));
    }
}
```

### Kullanımı
```csharp
AuthService auth = new AuthService();

// Audit log
auth.UserLoggedIn += (s, e) =>
    Console.WriteLine($"[Log] {e.Timestamp:HH:mm:ss} - {e.Username} giriş yaptı. IP: {e.IpAddress}");

// Güvenlik sistemi — şüpheli IP kontrolü
auth.UserLoggedIn += (s, e) =>
{
    if (e.IpAddress.StartsWith("192.168."))
        Console.WriteLine($"[Güvenlik] {e.Username} iç ağdan bağlandı.");
    else
        Console.WriteLine($"[Güvenlik] {e.Username} dış ağdan bağlandı — doğrulama kodu gönderildi.");
};

// Çıkış logu
auth.UserLoggedOut += (s, e) =>
    Console.WriteLine($"[Log] {e.Timestamp:HH:mm:ss} - {e.Username} oturumu sonlandırdı.");

auth.Login("ahmet", "sifre123", "85.100.22.5");
// ahmet girişi başarılı.
// [Log] 14:32:10 - ahmet giriş yaptı. IP: 85.100.22.5
// [Güvenlik] ahmet dış ağdan bağlandı — doğrulama kodu gönderildi.

auth.Logout("ahmet", "85.100.22.5");
// ahmet çıkış yaptı.
// [Log] 14:45:03 - ahmet oturumu sonlandırdı.
```

## Delegate vs Event Farkı

```csharp
public class Comparison
{
    public delegate void AlertHandler(string message);

    // Delegate: dışarıdan hem atanabilir hem çağrılabilir
    public AlertHandler? OnAlertDelegate;

    // Event: dışarıdan yalnızca += / -= yapılabilir, doğrudan çağrılamaz
    public event AlertHandler? OnAlertEvent;

    public void TriggerBoth()
    {
        OnAlertDelegate?.Invoke("Delegate ile tetiklendi");
        OnAlertEvent?.Invoke("Event ile tetiklendi");
    }
}

Comparison c = new Comparison();

// Delegate: dışarıdan doğrudan atama ve çağırma mümkün
c.OnAlertDelegate = msg => Console.WriteLine(msg);
c.OnAlertDelegate?.Invoke("Dışarıdan çağrıldı"); // İzin verilir

// Event: dışarıdan yalnızca abone olunabilir
c.OnAlertEvent += msg => Console.WriteLine(msg);
// c.OnAlertEvent?.Invoke("..."); // DERLEME HATASI — event sınıf dışından çağrılamaz
// c.OnAlertEvent = null;         // DERLEME HATASI — event sınıf dışından atanamaz
```
