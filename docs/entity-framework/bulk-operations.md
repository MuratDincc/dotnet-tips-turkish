# Bulk Operations

Entity Framework Core, veritabanı işlemleri için güçlü bir araçtır ancak büyük veri kümelerinde tek tek işlem yapmak yavaş ve maliyetli olabilir. Bulk operations, performansı artırmak ve işlem sürelerini azaltmak için etkili bir yöntemdir. Ancak, yanlış kullanımı veri tutarsızlıklarına ve performans sorunlarına neden olabilir.

---

## 1. Tek Tek İşlem Yaparak Performansı Düşürmek

❌ **Yanlış Kullanım:** Büyük veri kümelerinde tek tek işlem yapmak.

```csharp
foreach (var user in users)
{
    user.IsActive = true;
    context.Users.Update(user);
    context.SaveChanges();
}
```

✅ **İdeal Kullanım:** Bulk update ile tüm verileri tek bir işlemde güncelleyin.

```csharp
context.Users
    .Where(u => !u.IsActive)
    .ExecuteUpdate(u => u.SetProperty(x => x.IsActive, true));
```

---

## 2. Gereksiz Veri Tabanı Tetikleyicileri Çalıştırmak

❌ **Yanlış Kullanım:** Bulk işlemleri tetikleyicilerle birleştirmek.

```csharp
foreach (var product in products)
{
    product.StockCount += 10;
    context.Products.Update(product);
    context.SaveChanges(); // Her işlemde tetikleyici çalışır
}
```

✅ **İdeal Kullanım:** Bulk işlemler tetikleyicilerden bağımsız olarak yürütülmelidir.

```csharp
context.Products
    .Where(p => p.StockCount > 0)
    .ExecuteUpdate(p => p.SetProperty(x => x.StockCount, x => x.StockCount + 10));
```

---

## 3. Gereksiz Büyük Veri Setlerini Belleğe Yüklemek

❌ **Yanlış Kullanım:** Verileri önce belleğe yükleyip sonra güncellemek.

```csharp
var products = context.Products.ToList();
foreach (var product in products)
{
    product.Price += 5;
}
context.SaveChanges();
```

✅ **İdeal Kullanım:** Belleği verimli kullanarak doğrudan veritabanı üzerinde işlem yapın.

```csharp
context.Products
    .Where(p => p.Price < 100)
    .ExecuteUpdate(p => p.SetProperty(x => x.Price, x => x.Price + 5));
```

---

## 4. Bulk Delete İşlemlerinde Yanlış Filtreleme

❌ **Yanlış Kullanım:** Yanlış filtrelerle gereksiz verileri silmek.

```csharp
context.Products.RemoveRange(context.Products.Where(p => p.IsDiscontinued));
context.SaveChanges();
```

✅ **İdeal Kullanım:** Bulk delete ile doğrudan hedeflenen veriyi kaldırın.

```csharp
context.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteDelete();
```

---

## 5. Transaction Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Büyük işlemleri transaction olmadan gerçekleştirmek.

```csharp
context.Products
    .Where(p => p.CategoryId == 1)
    .ExecuteDelete();
context.Users
    .Where(u => u.IsInactive)
    .ExecuteUpdate(u => u.SetProperty(x => x.IsInactive, false));
```

✅ **İdeal Kullanım:** Tüm işlemleri bir transaction içinde yürütün.

```csharp
using var transaction = context.Database.BeginTransaction();
context.Products
    .Where(p => p.CategoryId == 1)
    .ExecuteDelete();
context.Users
    .Where(u => u.IsInactive)
    .ExecuteUpdate(u => u.SetProperty(x => x.IsInactive, false));
transaction.Commit();
```

---

## 6. Performans Optimizasyonu İçin Dış Kütüphaneleri Göz Ardı Etmek

❌ **Yanlış Kullanım:** Performans için yalnızca Entity Framework yöntemlerine güvenmek.

```csharp
context.BulkInsert(data); // EF'nin dahili yöntemleri
```

✅ **İdeal Kullanım:** Performansı artırmak için ek kütüphanelerden yararlanın.

```csharp
context.BulkInsert(data, options =>
{
    options.BatchSize = 1000;
    options.EnableStreaming = true;
});
```