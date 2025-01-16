# Multiple Mapping

Dapper, birden fazla nesneyi tek bir sorgudan eşleştirmek için **Multiple Mapping** özelliği sunar. Bu, özellikle ilişkisel veritabanı sorgularında, birden fazla tablodan veri çekme işlemlerinde oldukça kullanışlıdır. Ancak, yanlış kullanıldığında karmaşık kod yapısına ve performans kaybına neden olabilir.

---

## 1. Tek Nesne Eşleştirme

❌ **Yanlış Kullanım:** Tabloları manuel olarak birleştirme.

```csharp
var query = "SELECT * FROM Orders o JOIN Customers c ON o.CustomerId = c.Id";
var data = connection.Query(query).ToList(); // Eşleme yapılmaz
```

✅ **İdeal Kullanım:** Tek bir sorgudan birden fazla nesneyi eşleyin.

```csharp
var query = "SELECT o.*, c.* FROM Orders o JOIN Customers c ON o.CustomerId = c.Id";
var result = connection.Query<Order, Customer, Order>(
    query,
    (order, customer) =>
    {
        order.Customer = customer;
        return order;
    },
    splitOn: "Id"
).ToList();
```

---

## 2. Birden Fazla Nesne ve Karmaşık Eşleme

Dapper, birden fazla nesneyi eşlemek için `splitOn` özelliğini kullanır. Bu, her nesne için ayırıcı sütunlar belirlemenize olanak tanır.

✅ **Örnek:**

```csharp
var query = @"
    SELECT o.Id, o.OrderDate, c.Id, c.Name, p.Id, p.Name 
    FROM Orders o 
    JOIN Customers c ON o.CustomerId = c.Id
    JOIN Products p ON o.ProductId = p.Id";

var result = connection.Query<Order, Customer, Product, Order>(
    query,
    (order, customer, product) =>
    {
        order.Customer = customer;
        order.Product = product;
        return order;
    },
    splitOn: "Id,Id"
).ToList();
```

---

## 3. Performans İpuçları

- **splitOn Kullanımı:** `splitOn` özelliğini doğru belirleyin; aksi halde yanlış eşleme yapılabilir.
- **Gereksiz Verileri Dahil Etmeyin:** Sadece ihtiyaç duyulan sütunları seçin.
- **İlişki Yönetimi:** Nesneler arası ilişkileri kod tarafında yönetin.

---

## 4. Çoklu Eşleme ile Sorun Giderme

- **Doğru splitOn Ayarı:** `splitOn` özelliğinde sütun adlarının sırasını kontrol edin.
- **Sütun İsimlendirme Çakışmaları:** Eğer aynı isimde sütunlar varsa, alias (takma ad) kullanın.

```sql
SELECT 
    o.Id AS OrderId, 
    c.Id AS CustomerId, 
    p.Id AS ProductId 
FROM Orders o 
JOIN Customers c ON o.CustomerId = c.Id
JOIN Products p ON o.ProductId = p.Id
```