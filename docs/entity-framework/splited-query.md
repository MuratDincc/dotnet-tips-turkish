# Split Query

Entity Framework Core'da Split Query, karmaşık sorguları birden çok SQL sorgusuna bölerek performans iyileştirmesi yapar. Bu özellik, özellikle büyük veri kümelerinde veritabanı işlemlerini optimize etmek için kullanılır. Ancak, yanlış kullanımı hem performans hem de veri tutarlılığı sorunlarına yol açabilir.

---

## 1. Split Query'yi Gereksiz Kullanmak

❌ **Yanlış Kullanım:** Split Query'yi küçük veri setlerinde kullanmak.

```csharp
var users = context.Users
    .Include(u => u.Orders)
    .AsSplitQuery()
    .ToList();
```

✅ **İdeal Kullanım:** Split Query'yi yalnızca büyük veri kümeleri için kullanın.

```csharp
var users = context.Users
    .Include(u => u.Orders)
    .AsSplitQuery()
    .Where(u => u.IsActive)
    .ToList();
```

---

## 2. Performans Avantajlarını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Split Query kullanmadan büyük veri setlerini tek bir sorguda yüklemek.

```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Products)
    .ToList();
```

✅ **İdeal Kullanım:** Büyük veri setlerini Split Query ile bölerek işleyin.

```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Products)
    .AsSplitQuery()
    .ToList();
```

---

## 3. Split Query ve Track Değişikliklerini Yanlış Yönetmek

❌ **Yanlış Kullanım:** Split Query kullanırken veritabanı değişikliklerini yanlış anlamak.

```csharp
var customers = context.Customers
    .Include(c => c.Orders)
    .AsSplitQuery()
    .ToList();

customers[0].Name = "Updated Name";
context.SaveChanges(); // Beklenmeyen sonuçlar
```

✅ **İdeal Kullanım:** Split Query'nin salt okunur işlemler için daha uygun olduğunu unutmayın.

```csharp
var customers = context.Customers
    .Include(c => c.Orders)
    .AsSplitQuery()
    .ToList();

// Değişiklik yapmak yerine yeni bir işlem başlatın.
var customerToUpdate = customers.First();
customerToUpdate.Name = "Updated Name";
context.Update(customerToUpdate);
context.SaveChanges();
```

---

## 4. Split Query'yi Tüm Sorgular İçin Varsayılan Yapmak

❌ **Yanlış Kullanım:** Tüm sorgularda Split Query kullanarak gereksiz sorgular oluşturmak.

```csharp
optionsBuilder.UseSqlServer(connectionString, b => b.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

✅ **İdeal Kullanım:** Split Query'yi yalnızca belirli sorgular için kullanın.

```csharp
var data = context.Data
    .Include(d => d.RelatedData)
    .AsSplitQuery()
    .ToList();
```

---

## 5. Veri Tutarlılığını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Split Query ile veri tutarlılığı sorunlarına neden olmak.

```csharp
var orders = context.Orders
    .Include(o => o.Products)
    .AsSplitQuery()
    .ToList();

// Diğer işlemler sırasında veri değişebilir.
```

✅ **İdeal Kullanım:** Veri tutarlılığı kritikse Split Query yerine tek sorgu kullanın.

```csharp
var orders = context.Orders
    .Include(o => o.Products)
    .ToList();
```