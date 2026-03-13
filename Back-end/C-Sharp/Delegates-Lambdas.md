# Delegate
Delegate, bir metodu referans olarak tutan tip-güvenli bir işaretçidir. Bir değişkenin sayı veya string tutması gibi, delegate de bir metodu tutabilir ve onu daha sonra çağırabilir. Metotların parametre olarak geçilebilmesini sağlar.

## Temel Tanım
Bir ödeme sisteminde farklı ödeme yöntemlerini (kredi kartı, havale, kripto) tek bir delegate üzerinden çalıştırmak:

```csharp
// 1. Delegate tipini tanımla — hangi imzadaki metotları tutacağını belirtir
public delegate bool PaymentProcessor(decimal amount);

// 2. Uygun imzalı metotlar
public static bool ChargeWithCreditCard(decimal amount)
{
    Console.WriteLine($"Kredi kartından {amount:C} tahsil edildi.");
    return true;
}

public static bool TransferWithBankWire(decimal amount)
{
    Console.WriteLine($"Havale ile {amount:C} gönderildi.");
    return true;
}

// 3. Delegate'e metot ata ve çağır
PaymentProcessor pay = ChargeWithCreditCard;
pay(250.00m); // Kredi kartından ₺250,00 tahsil edildi.

pay = TransferWithBankWire;
pay(250.00m); // Havale ile ₺250,00 gönderildi.

// .Invoke() ile de çağırabilirsiniz
pay.Invoke(250.00m);
```

## Multicast Delegates
Delegate'ler birden fazla metodu aynı anda tutabilir. Her metot sırasıyla çağrılır. Dönüş tipi varsa yalnızca son metodun değeri alınır.

Bir sipariş onaylandığında hem e-posta hem SMS göndermek:

```csharp
public delegate void OrderConfirmation(string customerName, string orderId);

void SendEmail(string name, string orderId)
    => Console.WriteLine($"[Email] {name}'e sipariş #{orderId} onay maili gönderildi.");

void SendSms(string name, string orderId)
    => Console.WriteLine($"[SMS] {name}'in telefonuna sipariş #{orderId} bildirimi gönderildi.");

OrderConfirmation notify = SendEmail;
notify += SendSms; // Her ikisi de çalışır

notify("Ahmet", "ORD-1042");
// [Email] Ahmet'e sipariş #ORD-1042 onay maili gönderildi.
// [SMS] Ahmet'in telefonuna sipariş #ORD-1042 bildirimi gönderildi.

// -= ile çıkarılabilir
notify -= SendSms;
notify("Mehmet", "ORD-1043"); // Artık yalnızca email gönderir
```

## Parametre Olarak Geçirme
Delegate'ler parametre olarak geçirilerek, bir metodun davranışını dışarıdan belirlemesine olanak tanır.

Bir kargo şirketinin farklı fiyatlandırma stratejileriyle kargo ücreti hesaplaması:

```csharp
public delegate decimal ShippingCalculator(decimal orderTotal, double weightKg);

public static decimal CalculateShipping(decimal total, double weight, ShippingCalculator calculator)
{
    decimal fee = calculator(total, weight);
    Console.WriteLine($"Kargo ücreti: {fee:C}");
    return fee;
}

// Standart kargo: ağırlığa göre
decimal StandardShipping(decimal total, double weight) => (decimal)weight * 4.50m;

// Ücretsiz kargo: 500₺ üzeri siparişlerde
decimal FreeOverThreshold(decimal total, double weight) => total >= 500 ? 0m : 29.90m;

CalculateShipping(320m, 2.5, StandardShipping);      // Kargo ücreti: ₺11,25
CalculateShipping(320m, 2.5, FreeOverThreshold);     // Kargo ücreti: ₺29,90
CalculateShipping(650m, 2.5, FreeOverThreshold);     // Kargo ücreti: ₺0,00
```

## Action Delegate
`void` dönen metotlar için hazır delegate türüdür. Kendi delegate tipini tanımlamak yerine doğrudan `Action` kullanılır.

```csharp
// Action<T> — tek parametre, void döner
Action<string> log = message => Console.WriteLine($"[LOG] {message}");
log("Kullanıcı giriş yaptı.");

// Action<T1, T2> — iki parametre, void döner
Action<string, decimal> recordTransaction = (description, amount) =>
    Console.WriteLine($"İşlem kaydedildi: {description} — {amount:C}");

recordTransaction("Alışveriş", 149.90m);
// İşlem kaydedildi: Alışveriş — ₺149,90
```

### Action'ı parametre olarak geçirmek
Bir sipariş listesinin her birini farklı biçimlerde işlemek:

```csharp
public class Order
{
    public string Id { get; set; }
    public string Customer { get; set; }
    public decimal Total { get; set; }
}

public static void ProcessOrders(List<Order> orders, Action<Order> handler)
{
    foreach (var order in orders)
        handler(order);
}

var orders = new List<Order>
{
    new Order { Id = "ORD-01", Customer = "Ahmet", Total = 299m },
    new Order { Id = "ORD-02", Customer = "Ayşe",  Total = 540m },
};

// Fatura yazdır
ProcessOrders(orders, o => Console.WriteLine($"Fatura: {o.Id} — {o.Customer}: {o.Total:C}"));

// Kargo etiketi oluştur
ProcessOrders(orders, o => Console.WriteLine($"Kargo etiketi: {o.Id} → {o.Customer}"));
```

## Func Delegate
Değer döndüren metotlar için hazır delegate türüdür. Son tür parametresi her zaman dönüş tipidir.

```csharp
// Func<TResult> — parametre yok, değer döner
Func<DateTime> getNow = () => DateTime.Now;
Console.WriteLine(getNow()); // 13.03.2026 14:22:00

// Func<TInput, TResult> — bir parametre alır, değer döner
Func<decimal, decimal> applyVat = price => price * 1.20m;
Console.WriteLine(applyVat(100m)); // 120

// Func<T1, T2, TResult> — iki parametre alır, değer döner
Func<decimal, int, decimal> applyDiscount = (price, percent) => price * (1 - percent / 100m);
Console.WriteLine(applyDiscount(200m, 15)); // 170
```

### Func'ı parametre olarak geçirmek
Ürün listesini farklı kurallara göre filtrelemek:

```csharp
public class Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Category { get; set; }
}

public static List<Product> FilterProducts(List<Product> products, Func<Product, bool> criteria)
{
    var result = new List<Product>();
    foreach (var p in products)
        if (criteria(p)) result.Add(p);
    return result;
}

var products = new List<Product>
{
    new Product { Name = "Laptop",    Price = 25000m, Category = "Elektronik" },
    new Product { Name = "Kulaklık",  Price = 850m,   Category = "Elektronik" },
    new Product { Name = "Sandalye",  Price = 1200m,  Category = "Mobilya"    },
    new Product { Name = "Monitör",   Price = 7500m,  Category = "Elektronik" },
};

// Yalnızca elektronik ürünler
var electronics = FilterProducts(products, p => p.Category == "Elektronik");

// 1000₺ altındaki ürünler
var affordable = FilterProducts(products, p => p.Price < 1000m);

// Elektronik ve 5000₺ altı
var budgetElectronics = FilterProducts(products, p => p.Category == "Elektronik" && p.Price < 5000m);
```

## Lambda Expressions
Lambda, isimsiz bir metot yazmanın kısa yoludur. Tam metot bildirmek yerine `=>` sözdizimini kullanarak satır içi mantık tanımlanır.

```csharp
// İfade lambda (expression lambda) — tek satır, return yazmaya gerek yok
Func<decimal, decimal> addVat  = price => price * 1.20m;
Func<string, string>   greet   = name  => $"Merhaba, {name}!";
Func<int, int, int>    max     = (a, b) => a > b ? a : b;

// Blok lambda (statement lambda) — birden fazla satır, return gerekli
Func<string, string> formatCard = cardNumber =>
{
    string digits = new string(cardNumber.Where(char.IsDigit).ToArray());
    return $"{digits[..4]} {digits[4..8]} {digits[8..12]} {digits[12..]}";
};

Console.WriteLine(formatCard("1234567890123456")); // 1234 5678 9012 3456
```

### Closure — Lambda'nın Dış Değişkeni Yakalaması
Lambda, tanımlandığı kapsamdaki değişkenlere erişebilir. Bu özellik closure olarak adlandırılır.

```csharp
decimal discountRate = 0.10m; // Dış değişken

Func<decimal, decimal> applyDiscount = price => price * (1 - discountRate);

Console.WriteLine(applyDiscount(500m)); // 450 (500 * 0.90)

// discountRate değişirse lambda da yeni değeri kullanır
discountRate = 0.25m;
Console.WriteLine(applyDiscount(500m)); // 375 (500 * 0.75)
```
