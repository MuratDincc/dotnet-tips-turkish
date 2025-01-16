# Async/Await ile Asenkron Sorgular

Dapper, asenkron sorgular için `async/await` desteği sunar. Asenkron programlama, yüksek performanslı ve ölçeklenebilir uygulamalar oluşturmanın temel taşlarından biridir. Ancak, asenkron işlemlerin yanlış kullanımı beklenmeyen davranışlara ve performans sorunlarına yol açabilir.

---

## 1. Temel Asenkron Sorgular

❌ **Yanlış Kullanım:** Asenkron sorguları senkron olarak çağırmak.

```csharp
var query = "SELECT * FROM Users";
var users = connection.QueryAsync<User>(query).Result; // Deadlock riski
```

✅ **İdeal Kullanım:** Asenkron çağrıları her zaman `await` ile bekleyin.

```csharp
var query = "SELECT * FROM Users";
var users = await connection.QueryAsync<User>(query);
```

---

## 2. Birden Fazla Asenkron Sorgu Yönetimi

❌ **Yanlış Kullanım:** Sorguları sırayla çalıştırmak.

```csharp
var orders = await connection.QueryAsync<Order>("SELECT * FROM Orders");
var customers = await connection.QueryAsync<Customer>("SELECT * FROM Customers");
```

✅ **İdeal Kullanım:** Sorguları aynı anda başlatın ve `Task.WhenAll` ile bekleyin.

```csharp
var ordersTask = connection.QueryAsync<Order>("SELECT * FROM Orders");
var customersTask = connection.QueryAsync<Customer>("SELECT * FROM Customers");

await Task.WhenAll(ordersTask, customersTask);

var orders = ordersTask.Result;
var customers = customersTask.Result;
```

---

## 3. Transaction ile Asenkron İşlemler

Asenkron işlemleri transaction ile birleştirmek mümkündür.

✅ **Örnek:**

```csharp
using var transaction = connection.BeginTransaction();

try
{
    var insertQuery = "INSERT INTO Orders (CustomerId) VALUES (@CustomerId)";
    await connection.ExecuteAsync(insertQuery, new { CustomerId = 1 }, transaction);

    var updateQuery = "UPDATE Customers SET IsActive = @IsActive WHERE Id = @Id";
    await connection.ExecuteAsync(updateQuery, new { IsActive = true, Id = 1 }, transaction);

    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```

---

## 4. Performans ve Kaynak Yönetimi

- **Connection Pooling:** Asenkron işlemlerde bağlantı havuzunun verimli kullanıldığından emin olun.
- **Cancellation Token Kullanımı:** Uzun süren işlemler için iptal mekanizmaları kullanın.

✅ **Örnek:**

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

var query = "SELECT * FROM Orders WHERE OrderDate > @Date";
var orders = await connection.QueryAsync<Order>(query, new { Date = DateTime.UtcNow.AddDays(-30) }, cancellationToken: cts.Token);
```

---

## 5. Deadlock Sorunlarını Önlemek

❌ **Yanlış Kullanım:** `.Result` veya `.Wait()` kullanmak.

```csharp
var query = "SELECT * FROM Products";
var products = connection.QueryAsync<Product>(query).Result; // Deadlock riski
```

✅ **İdeal Kullanım:** `await` anahtar kelimesini kullanın.

```csharp
var query = "SELECT * FROM Products";
var products = await connection.QueryAsync<Product>(query);
```