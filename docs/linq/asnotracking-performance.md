# AsNoTracking Kullanımı ile Performans Artışı

Entity Framework, veritabanı sorgularını takip etmek ve değişiklikleri izlemek için varsayılan olarak bir **tracking** mekanizması kullanır. Ancak, yalnızca okuma işlemleri için bu izleme mekanizması gereksizdir ve performans kaybına neden olabilir. `AsNoTracking`, izleme mekanizmasını devre dışı bırakarak bu sorunu çözmek için kullanılır.

---

## 1. AsNoTracking Nedir?

`AsNoTracking`, sorgu sonucunda dönen nesnelerin değişiklik izlenmesini devre dışı bırakan bir Entity Framework özelliğidir. Özellikle sadece okuma işlemlerinde bu yöntem, bellek ve işlemci kullanımını azaltarak performansı artırır.

---

## 2. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** Tracking'in devrede olduğu durumlarda gereksiz bellek kullanımı.

❌ **Yanlış Kullanım:**

```csharp
using var context = new AppDbContext();

var customers = await context.Customers
    .Where(c => c.IsActive)
    .ToListAsync();

foreach (var customer in customers)
{
    Console.WriteLine(customer.Name);
}
```

Bu sorgu, dönen `customers` listesindeki her bir nesne için izleme bilgisi tutar. Ancak izleme, yalnızca okuma işlemlerinde gereksizdir.

---

### **İdeal Kullanım:** `AsNoTracking` ile gereksiz izlemeyi devre dışı bırakmak.

✅ **İdeal Kullanım:**

```csharp
using var context = new AppDbContext();

var customers = await context.Customers
    .AsNoTracking()
    .Where(c => c.IsActive)
    .ToListAsync();

foreach (var customer in customers)
{
    Console.WriteLine(customer.Name);
}
```

Bu yöntem, yalnızca okuma işlemleri için kullanıldığından izleme mekanizması devre dışı bırakılarak performans artışı sağlanır.

---

## 3. Performans Kazanımı

`AsNoTracking`, özellikle aşağıdaki durumlarda performansı artırır:
- Büyük veri kümelerinde sorgular çalıştırılırken.
- Sorgular sadece veri okuma amacıyla kullanıldığında.
- Birden fazla sorgu aynı anda çalıştırıldığında.

Ölçekli sistemlerde bu performans farkı ciddi ölçüde hissedilir.

---

## 4. `AsNoTrackingWithIdentityResolution` Kullanımı

`AsNoTracking` ile birlikte aynı kimliğe sahip nesnelerin doğru şekilde çözülmesi için `AsNoTrackingWithIdentityResolution` kullanılabilir.

✅ **Örnek:**

```csharp
using var context = new AppDbContext();

var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer)
    .ToListAsync();

foreach (var order in orders)
{
    Console.WriteLine($"Order ID: {order.Id}, Customer: {order.Customer.Name}");
}
```

Bu yöntem, ilişkisel verilerle çalışırken izleme mekanizması olmadan aynı kimlikteki nesneleri birleştirir.

---

## 5. Örnek Performans Testi

Aşağıdaki örnek, `AsNoTracking` ile standart sorgular arasındaki performans farkını gösterir:

```csharp
var stopwatch = Stopwatch.StartNew();

using var context = new AppDbContext();

// AsNoTracking olmadan
var trackedCustomers = await context.Customers
    .Where(c => c.IsActive)
    .ToListAsync();

stopwatch.Stop();
Console.WriteLine($"Tracked Query Time: {stopwatch.ElapsedMilliseconds} ms");

stopwatch.Restart();

// AsNoTracking ile
var untrackedCustomers = await context.Customers
    .AsNoTracking()
    .Where(c => c.IsActive)
    .ToListAsync();

stopwatch.Stop();
Console.WriteLine($"Untracked Query Time: {stopwatch.ElapsedMilliseconds} ms");
```

Sonuçlar, `AsNoTracking` kullanımıyla daha kısa sorgu süreleri ve daha az bellek kullanımı sağlayacaktır.