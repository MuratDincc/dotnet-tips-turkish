# Performans Optimizasyonu

Performans optimizasyonu, özellikle büyük ölçekli uygulamalarda kaynak tüketimini azaltmak ve kullanıcı deneyimini iyileştirmek için hayati öneme sahiptir. Yanlış uygulamalar sistemi yavaşlatabilir, hatta çökmesine neden olabilir. Bu rehber, Entity Framework Core'da performansı artırmaya yönelik en iyi uygulamaları ve yaygın hataları ele alır.

---

## 1. Gereksiz Veri Getirme

❌ **Yanlış Kullanım:** Gereksiz tüm sütunları sorguya dahil etmek.

```csharp
var products = context.Products.ToList(); // Tüm sütunlar getirilir
```

✅ **İdeal Kullanım:** Sadece gerekli sütunları projekte edin.

```csharp
var productNames = context.Products
    .Select(p => new { p.Name, p.Price })
    .ToList();
```

---

## 2. Eager Loading ve Lazy Loading'i Yanlış Yönetmek

❌ **Yanlış Kullanım:** Gereksiz `Include` kullanımıyla fazla veri yüklemek.

```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Products)
    .ToList();
```

✅ **İdeal Kullanım:** İhtiyaç duyulan veriler için `Include` kullanın.

```csharp
var orders = context.Orders
    .Include(o => o.Products)
    .Where(o => o.OrderDate > DateTime.UtcNow.AddMonths(-1))
    .ToList();
```

---

## 3. N+1 Sorgu Problemini Göz Ardı Etmek

❌ **Yanlış Kullanım:** İlgili veriler için ayrı sorgular çalıştırmak.

```csharp
var customers = context.Customers.ToList();
foreach (var customer in customers)
{
    var orders = context.Orders.Where(o => o.CustomerId == customer.Id).ToList();
}
```

✅ **İdeal Kullanım:** İlgili verileri tek bir sorguda getirin.

```csharp
var customersWithOrders = context.Customers
    .Include(c => c.Orders)
    .ToList();
```

---

## 4. Filtrelemeyi Bellek Tarafında Yapmak

❌ **Yanlış Kullanım:** Tüm veriyi bellek içinde filtrelemek.

```csharp
var orders = context.Orders.ToList();
var recentOrders = orders.Where(o => o.OrderDate > DateTime.UtcNow.AddMonths(-1));
```

✅ **İdeal Kullanım:** Filtrelemeyi veritabanı tarafında yapın.

```csharp
var recentOrders = context.Orders
    .Where(o => o.OrderDate > DateTime.UtcNow.AddMonths(-1))
    .ToList();
```

---

## 5. Indeksleri Göz Ardı Etmek

❌ **Yanlış Kullanım:** Sorgular için uygun indekslerin olmaması.

```sql
SELECT * FROM Orders WHERE CustomerId = 123;
-- Bu sorgu indeks yoksa yavaş çalışabilir.
```

✅ **İdeal Kullanım:** Sık kullanılan sütunlar için indeksler oluşturun.

```sql
CREATE INDEX IX_Orders_CustomerId ON Orders (CustomerId);
```

---

## 6. Çok Fazla İzleme (Tracking)

❌ **Yanlış Kullanım:** Gereksiz izleme (tracking) ile kaynak tüketimini artırmak.

```csharp
var products = context.Products.ToList(); // Tracking açık
products[0].Price = 10; // Değişiklik takibi yapar
```

✅ **İdeal Kullanım:** Sorguları izleme olmadan çalıştırın.

```csharp
var products = context.Products.AsNoTracking().ToList();
```

---

## 7. Transaction Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Çoklu veri tabanı işlemlerini transaction olmadan gerçekleştirmek.

```csharp
context.Products.Add(new Product { Name = "Product1" });
context.SaveChanges();
context.Orders.Add(new Order { ProductId = 1, Quantity = 10 });
context.SaveChanges();
```

✅ **İdeal Kullanım:** Tüm işlemleri bir transaction içinde yürütün.

```csharp
using var transaction = context.Database.BeginTransaction();
context.Products.Add(new Product { Name = "Product1" });
context.SaveChanges();
context.Orders.Add(new Order { ProductId = 1, Quantity = 10 });
context.SaveChanges();
transaction.Commit();
```