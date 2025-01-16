# Transaction Yönetimi

Dapper, birden fazla veri tabanı işlemini bir arada yönetmek için transaction desteği sunar. Transaction yönetimi, işlemlerin tamamı başarılı olmadığında yapılan değişikliklerin geri alınmasını sağlar ve veri tutarlılığını korur. Ancak, yanlış transaction yönetimi veri kayıplarına ve tutarsızlıklara neden olabilir.

---

## 1. Transaction Başlatma ve Kullanımı

❌ **Yanlış Kullanım:** Transaction kullanmadan birden fazla işlem gerçekleştirmek.

```csharp
connection.Execute("INSERT INTO Orders (CustomerId) VALUES (@CustomerId)", new { CustomerId = 1 });
connection.Execute("INSERT INTO OrderDetails (OrderId, ProductId) VALUES (@OrderId, @ProductId)", new { OrderId = 1, ProductId = 1 });
```

✅ **İdeal Kullanım:** Tüm işlemleri bir transaction içinde yönetin.

```csharp
using var transaction = connection.BeginTransaction();

try
{
    connection.Execute("INSERT INTO Orders (CustomerId) VALUES (@CustomerId)", new { CustomerId = 1 }, transaction);
    connection.Execute("INSERT INTO OrderDetails (OrderId, ProductId) VALUES (@OrderId, @ProductId)", new { OrderId = 1, ProductId = 1 }, transaction);
    
    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

---

## 2. Nested Transaction Kullanımı

❌ **Yanlış Kullanım:** Tek bir transaction içinde birden fazla işlem başlatmak.

```csharp
var transaction1 = connection.BeginTransaction();
var transaction2 = connection.BeginTransaction(); // Hatalı kullanım
```

✅ **İdeal Kullanım:** Her bağlantı için yalnızca bir transaction başlatın.

```csharp
using var transaction = connection.BeginTransaction();
connection.Execute("INSERT INTO Orders (CustomerId) VALUES (@CustomerId)", new { CustomerId = 1 }, transaction);
transaction.Commit();
```

---

## 3. Performans ve Güvenlik İpuçları

- **Transaction Commit Kontrolü:** Sadece tüm işlemler başarılı olduğunda `Commit` çağırın.
- **Timeout Ayarı:** Uzun süren transaction'lar için bir timeout ayarlayın.
- **Transaction Kapsamını Daraltın:** Transaction içinde sadece gerekli işlemleri yapın.

```csharp
using var transaction = connection.BeginTransaction(System.Data.IsolationLevel.Serializable);
try
{
    // İşlemler
    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

---

## 4. Isolation Levels (İzolasyon Seviyeleri)

Dapper, farklı izolasyon seviyeleriyle çalışabilir. Bu seviyeler, transaction'ın diğer transaction'larla nasıl etkileşime gireceğini belirler.

### İzolasyon Seviyeleri:
- **Read Uncommitted:** Diğer transaction'ların henüz commit edilmemiş verilerini okuyabilir.
- **Read Committed:** Sadece commit edilmiş veriler okunabilir.
- **Repeatable Read:** Bir transaction boyunca aynı veriyi okuma garantisi verir.
- **Serializable:** En yüksek izolasyon seviyesidir, veri tabanı kilitlemelerini artırabilir.

```csharp
using var transaction = connection.BeginTransaction(System.Data.IsolationLevel.RepeatableRead);
```