# Query Parameters Kullanımı

Dapper, SQL sorgularında parametre kullanımıyla hem güvenli hem de okunabilir kod yazmanızı sağlar. Parametreler sayesinde SQL enjeksiyonunu önleyebilir ve performansı artırabilirsiniz. Ancak, parametrelerin yanlış kullanımı performans kaybına ve güvenlik açıklarına neden olabilir.

---

## 1. Parametresiz Sorgular Yazmak

❌ **Yanlış Kullanım:** Parametresiz sorgular SQL enjeksiyonuna açık kapı bırakır.

```csharp
var query = $"SELECT * FROM Users WHERE Name = '{name}'";
var users = connection.Query<User>(query);
```

✅ **İdeal Kullanım:** Parametreler ile güvenli sorgular yazın.

```csharp
var query = "SELECT * FROM Users WHERE Name = @Name";
var users = connection.Query<User>(query, new { Name = name });
```

---

## 2. Birden Fazla Parametre Kullanımı

❌ **Yanlış Kullanım:** Parametreleri elle birleştirmek.

```csharp
var query = $"SELECT * FROM Users WHERE Name = '{name}' AND Age = {age}";
var users = connection.Query<User>(query);
```

✅ **İdeal Kullanım:** Dapper'ın parametre yönetimini kullanın.

```csharp
var query = "SELECT * FROM Users WHERE Name = @Name AND Age = @Age";
var users = connection.Query<User>(query, new { Name = name, Age = age });
```

---

## 3. Dinamik Parametrelerle Çalışmak

Dapper, dinamik parametreler ile esnek sorgular yazmanıza olanak tanır.

✅ **Örnek:**

```csharp
var parameters = new DynamicParameters();
parameters.Add("Name", name);
parameters.Add("Age", age);

var query = "SELECT * FROM Users WHERE Name = @Name AND Age = @Age";
var users = connection.Query<User>(query, parameters);
```

---

## 4. Performans ve Güvenlik İpuçları

- **SQL Enjeksiyonunu Önleyin:** Her zaman parametreli sorguları kullanın.  
- **DynamicParameters Kullanımı:** Dinamik senaryolarda parametre yönetimini kolaylaştırır.  
- **Parametrelerin Türlerine Dikkat Edin:** SQL türleriyle uyumlu parametreler kullanın.

```csharp
var parameters = new DynamicParameters();
parameters.Add("IsActive", true, DbType.Boolean);
parameters.Add("JoinDate", DateTime.Now, DbType.DateTime);

var query = "SELECT * FROM Users WHERE IsActive = @IsActive AND JoinDate > @JoinDate";
var users = connection.Query<User>(query, parameters);
```