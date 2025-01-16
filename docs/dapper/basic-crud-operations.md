# Basic CRUD İşlemleri

Dapper, basit ve performanslı veri erişimi sağlamak için kullanılan bir kütüphanedir. CRUD (Create, Read, Update, Delete) işlemleri Dapper ile oldukça kolaydır, ancak yanlış kullanımı güvenlik açıklarına ve performans sorunlarına neden olabilir.

---

## 1. Veri Ekleme (Insert)

❌ **Yanlış Kullanım:** SQL injection açık sorgular yazmak.

```csharp
var query = $"INSERT INTO Users (Name, Age) VALUES ('{name}', {age})";
connection.Execute(query);
```

✅ **İdeal Kullanım:** Parametreleri güvenli bir şekilde kullanın.

```csharp
var query = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
connection.Execute(query, new { Name = name, Age = age });
```

---

## 2. Veri Okuma (Read)

❌ **Yanlış Kullanım:** Tüm tabloyu belleğe yüklemek.

```csharp
var query = "SELECT * FROM Users";
var users = connection.Query<User>(query).ToList();
```

✅ **İdeal Kullanım:** Filtreleme ve projeksiyon ile daha az veri getirin.

```csharp
var query = "SELECT Id, Name FROM Users WHERE Age > @Age";
var users = connection.Query<User>(query, new { Age = 25 }).ToList();
```

---

## 3. Veri Güncelleme (Update)

❌ **Yanlış Kullanım:** Tüm sütunları gereksiz yere güncellemek.

```csharp
var query = "UPDATE Users SET Name = 'UpdatedName', Age = 30 WHERE Id = @Id";
connection.Execute(query, new { Id = userId });
```

✅ **İdeal Kullanım:** Yalnızca gerekli sütunları güncelleyin.

```csharp
var query = "UPDATE Users SET Name = @Name WHERE Id = @Id";
connection.Execute(query, new { Name = "UpdatedName", Id = userId });
```

---

## 4. Veri Silme (Delete)

❌ **Yanlış Kullanım:** Koşul olmadan veri silmek.

```csharp
var query = "DELETE FROM Users";
connection.Execute(query);
```

✅ **İdeal Kullanım:** Silme işlemlerini her zaman filtreleyin.

```csharp
var query = "DELETE FROM Users WHERE Id = @Id";
connection.Execute(query, new { Id = userId });
```

---

## 5. Performans ve Güvenlik İpuçları

- **Prepared Statements:** Her zaman parametreli sorguları kullanarak SQL enjeksiyonunu önleyin.  
- **Minimal Veri Getirme:** Tüm tabloyu belleğe yüklemek yerine ihtiyacınız olan sütunları seçin.  
- **Transaction Kullanımı:** Birden fazla işlemi birlikte yönetmek için transaction kullanmayı düşünün.

```csharp
using var transaction = connection.BeginTransaction();
try
{
    var insertQuery = "INSERT INTO Users (Name, Age) VALUES (@Name, @Age)";
    connection.Execute(insertQuery, new { Name = "Murat", Age = 33 }, transaction);

    var updateQuery = "UPDATE Users SET Age = @Age WHERE Name = @Name";
    connection.Execute(updateQuery, new { Name = "Murat", Age = 34 }, transaction);

    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```