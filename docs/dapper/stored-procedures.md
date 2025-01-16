# Stored Procedure Kullanımı

Dapper, stored procedure’ler ile etkili ve performanslı veri tabanı işlemleri gerçekleştirmenize olanak tanır. Stored procedure'ler, özellikle karmaşık veri tabanı işlemleri ve çoklu sorgular için tercih edilir. Ancak, doğru kullanılmadığında performans sorunlarına ve yönetim zorluklarına yol açabilir.

---

## 1. Stored Procedure Çağırma

❌ **Yanlış Kullanım:** Stored procedure'ü doğrudan metin olarak çağırmak.

```csharp
var query = "EXEC GetUsers";
var users = connection.Query<User>(query);
```

✅ **İdeal Kullanım:** `CommandType.StoredProcedure` kullanarak prosedürü çağırın.

```csharp
var users = connection.Query<User>("GetUsers", commandType: CommandType.StoredProcedure);
```

---

## 2. Parametreli Stored Procedure Kullanımı

❌ **Yanlış Kullanım:** Parametreleri elle birleştirmek.

```csharp
var query = $"EXEC GetUsersByAge {age}";
var users = connection.Query<User>(query);
```

✅ **İdeal Kullanım:** Parametreleri doğru şekilde tanımlayın.

```csharp
var users = connection.Query<User>(
    "GetUsersByAge",
    new { Age = age },
    commandType: CommandType.StoredProcedure);
```

---

## 3. Output Parametreleri Kullanımı

Dapper, stored procedure'den dönen `output` parametrelerini kolayca yönetebilir.

✅ **Örnek:**

```csharp
var parameters = new DynamicParameters();
parameters.Add("InputParam", 10);
parameters.Add("OutputParam", dbType: DbType.Int32, direction: ParameterDirection.Output);

connection.Execute("CalculateTotal", parameters, commandType: CommandType.StoredProcedure);

var total = parameters.Get<int>("OutputParam");
Console.WriteLine($"Total: {total}");
```

---

## 4. Çoklu Sonuç Kümesi (Multiple Result Sets)

Dapper, stored procedure'lerden dönen birden fazla sonuç kümesini de destekler.

✅ **Örnek:**

```csharp
using var multi = connection.QueryMultiple("GetUsersAndOrders", commandType: CommandType.StoredProcedure);

var users = multi.Read<User>().ToList();
var orders = multi.Read<Order>().ToList();
```

---

## 5. Performans ve Güvenlik İpuçları

- **Parametre Kullanımı:** SQL enjeksiyonunu önlemek için her zaman parametreli sorguları tercih edin.
- **CommandType.StoredProcedure:** Prosedür çağrılarında bu parametreyi mutlaka ekleyin.
- **Index ve Query Planlarına Dikkat:** Stored procedure'lerinizin verimli çalıştığından emin olun.

```csharp
var parameters = new { StartDate = DateTime.UtcNow.AddDays(-30), EndDate = DateTime.UtcNow };
var results = connection.Query("GetReportData", parameters, commandType: CommandType.StoredProcedure);
```