# Bulk İşlemler ve Performans Artışı

Dapper, tek seferde büyük miktarda veri işlemleri (bulk operations) için etkili bir araçtır. Ancak, bu işlemleri doğru yapılandırmazsanız performans sorunlarına yol açabilirsiniz. Bulk işlemleri optimize etmek, uygulamanızın kaynak tüketimini azaltır ve işlem sürelerini kısaltır.

---

## 1. Tekli Ekleme ve Performans Problemi

❌ **Yanlış Kullanım:** Her ekleme işlemi için ayrı bir sorgu çalıştırmak.

```csharp
foreach (var user in users)
{
    connection.Execute("INSERT INTO Users (Name, Age) VALUES (@Name, @Age)", user);
}
```

Bu yöntem, büyük veri setlerinde veritabanına gereksiz sorgular göndererek performansı düşürür.

✅ **İdeal Kullanım:** Tüm veriyi tek bir işlemle ekleyin.

```csharp
var query = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
connection.Execute(query, users);
```

---

## 2. Bulk Update İşlemleri

❌ **Yanlış Kullanım:** Her bir güncelleme için ayrı bir sorgu.

```csharp
foreach (var user in users)
{
    connection.Execute("UPDATE Users SET Age = @Age WHERE Id = @Id", user);
}
```

✅ **İdeal Kullanım:** Tek bir sorguda toplu güncelleme.

```csharp
var query = @"
    UPDATE Users 
    SET Age = CASE Id 
        WHEN @Id1 THEN @Age1 
        WHEN @Id2 THEN @Age2 
    END
    WHERE Id IN (@Id1, @Id2)";

connection.Execute(query, new 
{
    Id1 = users[0].Id, Age1 = users[0].Age,
    Id2 = users[1].Id, Age2 = users[1].Age
});
```

---

## 3. Performans İçin Table-Valued Parameters (TVP)

Dapper, doğrudan TVP desteği sunmaz ancak SQL Server'da TVP kullanarak performansı artırabilirsiniz.

✅ **Örnek:**

```csharp
var dataTable = new DataTable();
dataTable.Columns.Add("Id", typeof(int));
dataTable.Columns.Add("Name", typeof(string));

foreach (var user in users)
{
    dataTable.Rows.Add(user.Id, user.Name);
}

using var connection = new SqlConnection(connectionString);
using var command = connection.CreateCommand();
command.CommandText = "dbo.BulkInsertUsers";
command.CommandType = CommandType.StoredProcedure;

var parameter = command.Parameters.AddWithValue("@Users", dataTable);
parameter.SqlDbType = SqlDbType.Structured;

connection.Open();
command.ExecuteNonQuery();
```

---

## 4. Performans İpuçları

- **Batching:** İşlemleri gruplara ayırarak sorgu sayısını azaltın.
- **Transaction Kullanımı:** Bulk işlemler için `Transaction` kullanarak veri tutarlılığını sağlayın.
- **Profiling ve İzleme:** SQL Server'da `Query Execution Plan` kullanarak sorgularınızı optimize edin.