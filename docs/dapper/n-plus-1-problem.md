# N+1 Problemi

N+1 problemi, ilişkisel veritabanı sorgularında sıkça karşılaşılan bir performans tuzağıdır. Özellikle Dapper gibi ORM araçlarıyla çalışırken, doğru sorgulama yapılmadığında veritabanına gereksiz bir şekilde çok fazla sorgu gönderilmesine yol açar. Bu durum, uygulamanızın performansını ciddi şekilde etkileyebilir.

---

## 1. N+1 Probleminin Tanımı

N+1 problemi, bir listeye ait her öğe için ayrı bir sorgu çalıştırıldığında ortaya çıkar. Bu, toplamda 1 ana sorgu ve N adet alt sorgu çalıştırılması anlamına gelir.

❌ **Yanlış Kullanım:** Listeyi döngü içinde sorgulamak.

```csharp
var orders = connection.Query<Order>("SELECT * FROM Orders").ToList();

foreach (var order in orders)
{
    order.Customer = connection.QuerySingle<Customer>(
        "SELECT * FROM Customers WHERE Id = @Id",
        new { Id = order.CustomerId });
}
}
```

Bu yöntem, veritabanına gereksiz yere çok sayıda sorgu gönderir ve performansı düşürür.

✅ **İdeal Kullanım:** `JOIN` kullanarak tüm verileri tek bir sorguda getirin.

```csharp
var query = @"
    SELECT o.*, c.* 
    FROM Orders o
    JOIN Customers c ON o.CustomerId = c.Id";

var orders = connection.Query<Order, Customer, Order>(
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

## 2. Lazy Loading Tuzağı

Lazy loading (tembel yükleme), genellikle N+1 problemini tetikleyen bir yöntemdir. Dapper, varsayılan olarak lazy loading desteklemez, ancak manuel olarak uygulanabilir. Bu işlem performans sorunlarına yol açabilir.

❌ **Yanlış Kullanım:** Her nesne için ayrı sorgu.

```csharp
var orders = connection.Query<Order>("SELECT * FROM Orders").ToList();

foreach (var order in orders)
{
    order.Customer = GetCustomerById(order.CustomerId);
}

Customer GetCustomerById(int customerId)
{
    return connection.QuerySingle<Customer>(
        "SELECT * FROM Customers WHERE Id = @Id",
        new { Id = customerId });
}
```

✅ **İdeal Kullanım:** Gerekli tüm verileri birleştirerek getirin.

```csharp
var query = @"
    SELECT o.Id, o.OrderDate, c.Id AS CustomerId, c.Name AS CustomerName
    FROM Orders o
    JOIN Customers c ON o.CustomerId = c.Id";

var orders = connection.Query<Order, Customer, Order>(
    query,
    (order, customer) =>
    {
        order.Customer = customer;
        return order;
    },
    splitOn: "CustomerId"
).ToList();
```

---

## 3. Performans ve Kaynak Kullanımı

- **JOIN Kullanımı:** N+1 problemini önlemek için veritabanı sorgularında `JOIN` kullanın.
- **Liste İşlemleri:** Tüm listeyi tek bir sorguyla alın, döngü içinde sorgu yapmaktan kaçının.
- **Profiling Araçları:** Sorgularınızın veritabanına kaç kez gittiğini izlemek için profil araçlarını kullanın.