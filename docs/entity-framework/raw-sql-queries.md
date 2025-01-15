# Raw SQL Queries

Entity Framework Core, veritabanı ile iletişim kurmak için LINQ tabanlı sorgular sunar. Ancak, bazı durumlarda ham SQL sorgularını kullanmak gerekebilir. Ham SQL sorguları, yüksek performans ve esneklik sağlasa da, yanlış kullanımları güvenlik açıklarına ve performans sorunlarına yol açabilir.

---

## 1. SQL Injection Riski

❌ **Yanlış Kullanım:** Dinamik olarak oluşturulan SQL ifadeleri.

```csharp
var username = "admin";
var password = "password123";
var query = $"SELECT * FROM Users WHERE Username = '{username}' AND Password = '{password}'";
var users = context.Users.FromSqlRaw(query).ToList();
```

✅ **İdeal Kullanım:** Parametreli sorgular kullanarak SQL Injection riskini önleyin.

```csharp
var username = "admin";
var password = "password123";
var users = context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Username = {username} AND Password = {password}")
    .ToList();
```

---

## 2. Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Gereksiz büyük veri setlerini yüklemek.

```csharp
var products = context.Products.FromSqlRaw("SELECT * FROM Products").ToList();
```

✅ **İdeal Kullanım:** Sorguyu filtreleyerek yalnızca gerekli verileri yükleyin.

```csharp
var products = context.Products
    .FromSqlRaw("SELECT * FROM Products WHERE IsActive = 1")
    .ToList();
```

---

## 3. Ham SQL Sorgularını Test Edilebilir Hale Getirmemek

❌ **Yanlış Kullanım:** Ham SQL sorgularını test edilebilirlikten yoksun bırakmak.

```csharp
var orders = context.Orders.FromSqlRaw("SELECT * FROM Orders WHERE OrderDate > GETDATE()").ToList();
```

✅ **İdeal Kullanım:** SQL sorgularını metodlara taşıyarak test edilebilir hale getirin.

```csharp
public IEnumerable<Order> GetRecentOrders(DateTime sinceDate)
{
    return context.Orders
        .FromSqlInterpolated($"SELECT * FROM Orders WHERE OrderDate > {sinceDate}")
        .ToList();
}
```

---

## 4. Güvenilir Kaynaklardan Gelen SQL Kullanımı

❌ **Yanlış Kullanım:** SQL ifadelerini doğrudan kod içine gömmek.

```csharp
var results = context.Database.ExecuteSqlRaw("DELETE FROM Logs WHERE LogDate < '2023-01-01'");
```

✅ **İdeal Kullanım:** SQL ifadelerini merkezi bir yere taşıyın veya saklı yordamları (stored procedure) kullanın.

```csharp
var results = context.Database.ExecuteSqlRaw("EXEC ClearOldLogs @Date = {0}", new[] { "2023-01-01" });
```

---

## 5. SQL Hatalarını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Hata kontrolü yapmamak.

```csharp
var data = context.Users.FromSqlRaw("SELECT * FROM NonExistentTable").ToList();
```

✅ **İdeal Kullanım:** SQL sorgularını hata yakalama mekanizmaları ile birlikte kullanın.

```csharp
try
{
    var data = context.Users.FromSqlRaw("SELECT * FROM NonExistentTable").ToList();
}
catch (Exception ex)
{
    Console.WriteLine($"SQL Error: {ex.Message}");
}
```

---

## 6. Ham SQL ve Entity Framework Özelliklerini Birlikte Kullanamamak

❌ **Yanlış Kullanım:** Ham SQL sorguları ile Entity Framework özelliklerini entegre etmemek.

```csharp
var orders = context.Orders.FromSqlRaw("SELECT * FROM Orders").ToList();
foreach (var order in orders)
{
    context.Entry(order).Collection(o => o.OrderItems).Load();
}
```

✅ **İdeal Kullanım:** Ham SQL sorguları ile Entity Framework ilişkilerini birleştirin.

```csharp
var orders = context.Orders
    .FromSqlRaw("SELECT * FROM Orders")
    .Include(o => o.OrderItems)
    .ToList();
```