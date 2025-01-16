# QueryMultiple

Dapper, bir sorgudan birden fazla sonuç kümesini almanıza olanak tanır. `QueryMultiple` metodu sayesinde, ilişkisel veritabanlarındaki karmaşık sorguları basitleştirebilir ve birden fazla tabloyu tek bir işlemde çekebilirsiniz. Ancak, bu yöntemin yanlış kullanımı performans sorunlarına ve veri uyumsuzluklarına yol açabilir.

---

## 1. Temel QueryMultiple Kullanımı

❌ **Yanlış Kullanım:** Sorgu sonuçlarını manuel olarak bölmek.

```csharp
var query = "SELECT * FROM Orders; SELECT * FROM Customers";
var orders = connection.Query<Order>("SELECT * FROM Orders").ToList();
var customers = connection.Query<Customer>("SELECT * FROM Customers").ToList();
```

✅ **İdeal Kullanım:** `QueryMultiple` ile her iki sonucu tek bir sorguda alın.

```csharp
var query = "SELECT * FROM Orders; SELECT * FROM Customers";

using var multi = connection.QueryMultiple(query);
var orders = multi.Read<Order>().ToList();
var customers = multi.Read<Customer>().ToList();
```

---

## 2. Parametreli Sorgularla QueryMultiple Kullanımı

✅ **Örnek:** Parametreleri kullanarak sonuç kümesini dinamikleştirin.

```csharp
var query = "SELECT * FROM Orders WHERE CustomerId = @CustomerId; SELECT * FROM Customers WHERE Id = @CustomerId";

using var multi = connection.QueryMultiple(query, new { CustomerId = 1 });
var orders = multi.Read<Order>().ToList();
var customer = multi.Read<Customer>().FirstOrDefault();
```

---

## 3. Çoklu Sonuç Kümesiyle Karmaşık Veri Eşleme

Dapper, `QueryMultiple` ile karmaşık veri yapılarını yönetmeyi kolaylaştırır.

✅ **Örnek:**

```csharp
var query = @"
    SELECT o.Id, o.OrderDate, c.Id AS CustomerId, c.Name AS CustomerName
    FROM Orders o
    JOIN Customers c ON o.CustomerId = c.Id;
    SELECT * FROM Products;";

using var multi = connection.QueryMultiple(query);
var orders = multi.Read<OrderWithCustomer>().ToList();
var products = multi.Read<Product>().ToList();
```

Burada `OrderWithCustomer` aşağıdaki gibi bir sınıf olabilir:

```csharp
public class OrderWithCustomer
{
    public int Id { get; set; }
    public DateTime OrderDate { get; set; }
    public Customer Customer { get; set; }
}
```

---

## 4. Performans İpuçları

- **Minimum Veri Çekme:** Sadece ihtiyaç duyulan sütunları sorgulayın.
- **Doğru Sıra:** `QueryMultiple` ile alınan sonuçları doğru sırada okuyun. Yanlış sıra hatalara yol açabilir.
- **Kaynak Yönetimi:** `QueryMultiple` sonuçları tamamlandıktan sonra `Dispose` çağırarak kaynakları serbest bırakmayı unutmayın.