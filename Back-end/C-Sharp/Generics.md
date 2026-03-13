# Generic Sınıflar (Generic Classes)
Generic sınıf, tip parametresi alarak farklı veri tipleriyle çalışabilen, yeniden kullanılabilir sınıflardır. Tip, sınıfı kullanırken belirlenir; bu sayede tip güvenliği korunurken kod tekrarından kaçınılmış olur. Örneğin BoxOfInt, BoxOfString vb. için ayrı sınıflar yazmak yerine, tek bir Box<T> sınıfı yazarsınız.

```csharp
// T, sınıfın tip parametresidir ve herhangi bir veri tipi olabilir.
public class Box<T>
{
    public T Value { get; set; }

    public Box(T value)
    {
        Value = value;
    }
}
```

## Generic Sınıfların Kullanımı
```csharp
// Box sınıfını farklı veri tipleriyle oluşturan örnekler:
Box<int> intBox = new Box<int>(123);
Box<string> stringBox = new Box<string>("Test");
Box<double> doubleBox = new Box<double>(3.14);

// Değerlere erişim (Strongly Typed: Derleyici, değişkenin tipini derleme zamanında bilir ve yanlış tip kullanımı kodu çalıştırmadan hata olarak işaretlenir.)
int number = intBox.Value; // 123
string text = stringBox.Value; // "Test"
double pi = doubleBox.Value; // 3.14
```

## Object ile Karşılaştırma (Strongly Typed vs. Weakly Typed)
```csharp
public class Box
{
    public object Value { get; set; }
}

Box box = new Box();
box.Value = 42;

// Derleyici tipin ne olduğunu bilmiyor. Cast yapmak zorundayız.
int number = (int)box.Value;
// Unboxing işlemi sırasında yanlış tip kullanımı runtime hatasına neden olur. Kod çalıştırılana kadar bu hatayı göremezsiniz.
string text = (string)box.Value;
```

## Kısıtlamalar (Constraints)
Generic constraints, tür argümanı olarak hangi türlerin kullanılabileceğini sınırlar. Generic türün belirli yeteneklere sahip olmasını sağlayarak, kod içinde belirli işlemleri güvenli bir şekilde kullanmanıza olanak tanır

where anahtar kelimesiyle tip parametresine şart koşulabilir:

Not: c#'ta where kısıtlamalarının belirli bir yazım sırası zorunluluğu vardır. class/struct -> interface -> new() şeklinde sıralanmalıdır.

```csharp
// Depoda saklanabilecek her ürünün uyması gereken arayüz
public interface IStorable
{
    string Name { get; set; }
    int Quantity { get; set; }
}

// IStorable'ı uygulayan somut ürün sınıfları
public class ElectronicProduct : IStorable
{
    public string Name { get; set; } = "Bilinmeyen Elektronik";
    public int Quantity { get; set; } = 0;
    public string WarrantyPeriod { get; set; } = "2 Yıl";
}

public class FoodProduct : IStorable
{
    public string Name { get; set; } = "Bilinmeyen Gıda";
    public int Quantity { get; set; } = 0;
    public DateTime ExpirationDate { get; set; } = DateTime.Now.AddMonths(6);
}

// Yalnızca IStorable uygulayan ve parametresiz constructor'ı olan sınıfları kabul eder.
// class  : Değer tipi değil, referans tipi olmalı
// IStorable : Name ve Quantity özelliklerine sahip olmalı
// new()  : Parametresiz constructor olmalı (boş slot oluşturabilmek için)
public class Warehouse<T> where T : class, IStorable, new()
{
    public string WarehouseName { get; set; }
    public List<T> Items { get; set; } = new List<T>();

    public Warehouse(string warehouseName)
    {
        WarehouseName = warehouseName;
    }

    public void AddItem(T item)
    {
        Items.Add(item);
        Console.WriteLine($"{item.Name} depoya eklendi. Toplam stok: {item.Quantity}");
    }

    // new() kısıtlaması sayesinde T'nin parametresiz constructor'ı olduğu garanti edilir.
    // Böylece depoya henüz bilgileri doldurulmamış boş bir slot açabiliriz.
    public T ReserveEmptySlot()
    {
        T emptySlot = new T();
        Items.Add(emptySlot);
        Console.WriteLine($"Boş slot ayrıldı. Slot adı: '{emptySlot.Name}'");
        return emptySlot;
    }

    public void ListItems()
    {
        Console.WriteLine($"--- {WarehouseName} Envanteri ---");
        foreach (var item in Items)
            Console.WriteLine($"  {item.Name}: {item.Quantity} adet");
    }
}
```

### Kullanımı
```csharp
// Elektronik deposu — sadece ElectronicProduct kabul eder
Warehouse<ElectronicProduct> electronicWarehouse = new Warehouse<ElectronicProduct>("Elektronik Deposu");

electronicWarehouse.AddItem(new ElectronicProduct { Name = "Laptop", Quantity = 50 });
electronicWarehouse.AddItem(new ElectronicProduct { Name = "Telefon", Quantity = 120 });

// Yeni bir elektronik ürün slotu ayır, bilgileri sonradan doldur
ElectronicProduct reserved = electronicWarehouse.ReserveEmptySlot();
reserved.Name = "Tablet";
reserved.Quantity = 30;

electronicWarehouse.ListItems();
// Çıktı:
// --- Elektronik Deposu Envanteri ---
//   Laptop: 50 adet
//   Telefon: 120 adet
//   Tablet: 30 adet

// Gıda deposu — sadece FoodProduct kabul eder
Warehouse<FoodProduct> foodWarehouse = new Warehouse<FoodProduct>("Gıda Deposu");
foodWarehouse.AddItem(new FoodProduct { Name = "Makarna", Quantity = 200 });

// Derleme hatası: ElectronicProduct, FoodProduct deposuna eklenemez
// foodWarehouse.AddItem(new ElectronicProduct()); // HATA!
```

## Generic Metotlar (Generic Methods)
Generic metotlar, ait oldukları herhangi bir sınıftan bağımsız olarak kendi tip parametrelerini tanımlayan metotlardır. Generic bir sınıf oluşturmadan tek bir işlem için tip esnekliğini ihtiyaç duyduğunuzda kullanışlıdırlar.

Generic metotlar çoğunlukla yardımcı/utility işlemler için yazılır. Utility metotlar da doğası gereği state tutmadığından static olmaya yatkındır.

Generic bir metodun içinde, gerçek tip hakkında bilgi alabilirsiniz:
```csharp
public static string GetTypeName<T>(T input)
{
    // typeof(T) ifadesi, T'nin gerçek tipini temsil eder. Örneğin, T int ise typeof(T) int tipini verir.
    return typeof(T).Name; // Int32, String, List`1 gibi tip isimleri döner.
}
```

Aşağıdaki örnek, API katmanında her endpoint'in döndürdüğü yanıtı standart bir zarfa (envelope) saran `ResponseFactory` utility sınıfını göstermektedir. Her endpoint farklı bir veri tipi döndürdüğünden generic metot kullanımı burada doğal bir çözümdür.

```csharp
public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public string? ErrorMessage { get; set; }
    public DateTime Timestamp { get; set; }
}

public static class ResponseFactory
{
    // Başarılı yanıt sarar — herhangi bir veri tipiyle çalışır
    public static ApiResponse<T> Ok<T>(T data)
    {
        return new ApiResponse<T>
        {
            Success = true,
            Data = data,
            Timestamp = DateTime.UtcNow
        };
    }

    // Hata yanıtı sarar — T tipi bilinmese de aynı zarf yapısı kullanılır
    public static ApiResponse<T> Fail<T>(string errorMessage)
    {
        return new ApiResponse<T>
        {
            Success = false,
            ErrorMessage = errorMessage,
            Timestamp = DateTime.UtcNow
        };
    }

    // Sayfalı liste yanıtlarını sarar
    public static ApiResponse<List<T>> OkList<T>(List<T> items)
    {
        return new ApiResponse<List<T>>
        {
            Success = true,
            Data = items,
            Timestamp = DateTime.UtcNow
        };
    }
}
```

### Kullanımı
```csharp
public class UserDto  { public int Id { get; set; } public string Name { get; set; } }
public class OrderDto { public int Id { get; set; } public decimal Total { get; set; } }

// Farklı veri tipleri için aynı factory metotları kullanılır
ApiResponse<UserDto> userResponse = ResponseFactory.Ok(new UserDto { Id = 1, Name = "Ahmet" });
// { Success: true, Data: { Id: 1, Name: "Ahmet" }, ... }

ApiResponse<OrderDto> orderResponse = ResponseFactory.Ok(new OrderDto { Id = 42, Total = 349.90m });
// { Success: true, Data: { Id: 42, Total: 349.90 }, ... }

ApiResponse<UserDto> notFound = ResponseFactory.Fail<UserDto>("Kullanıcı bulunamadı.");
// { Success: false, ErrorMessage: "Kullanıcı bulunamadı.", Data: null, ... }

var users = new List<UserDto> { new UserDto { Id = 1, Name = "Ahmet" }, new UserDto { Id = 2, Name = "Mehmet" } };
ApiResponse<List<UserDto>> listResponse = ResponseFactory.OkList(users);
// { Success: true, Data: [ {...}, {...} ], ... }
```

