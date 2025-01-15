# Async LINQ

Async LINQ işlemleri, özellikle veritabanı sorgularında IO bağlamlı işlemleri optimize etmek için kullanılır. Doğru bir şekilde uygulandığında performansı artırabilir ve kullanıcı deneyimini iyileştirebilir. Ancak, yanlış kullanım performans kaybına ve kaynakların yanlış yönetilmesine yol açabilir.

---

## 1. Senkron LINQ ile Bekleme Sürelerini Artırmak

❌ **Yanlış Kullanım:** Senkron LINQ metotları kullanarak IO bağlamlı işlemleri engellemek.

```csharp
var users = context.Users
    .Where(u => u.IsActive)
    .ToList(); // Senkron çağrı, bloklama yaratır.
```

✅ **İdeal Kullanım:** Asenkron LINQ metotları kullanarak bloklamayı önleyin.

```csharp
var users = await context.Users
    .Where(u => u.IsActive)
    .ToListAsync();
```

---

## 2. `ToListAsync` Kullanımını Gereksiz Yere Zincirlemek

❌ **Yanlış Kullanım:** `ToListAsync` çağrısını gereksiz yere başka işlemlerle zincirlemek.

```csharp
var users = (await context.Users.ToListAsync())
    .Where(u => u.IsActive);
```

✅ **İdeal Kullanım:** Filtreleri doğrudan asenkron sorguya dahil edin.

```csharp
var users = await context.Users
    .Where(u => u.IsActive)
    .ToListAsync();
```

---

## 3. Her Satırda `Await` Kullanarak Performansı Azaltmak

❌ **Yanlış Kullanım:** Her işlemde ayrı ayrı `await` kullanmak.

```csharp
var userList = new List<User>();
foreach (var userId in userIds)
{
    var user = await context.Users.FindAsync(userId);
    userList.Add(user);
}
```

✅ **İdeal Kullanım:** Paralel işlemleri bir arada çalıştırmak için `Task.WhenAll` kullanın.

```csharp
var tasks = userIds.Select(id => context.Users.FindAsync(id).AsTask());
var userList = await Task.WhenAll(tasks);
```

---

## 4. Asenkron Olmayan Veriler İçin `ToListAsync` Kullanımı

❌ **Yanlış Kullanım:** Bellekte olan verilerde asenkron çağrı kullanmak.

```csharp
var inMemoryList = new List<int> { 1, 2, 3 };
var result = await inMemoryList.ToListAsync(); // Geçersiz kullanım
```

✅ **İdeal Kullanım:** Asenkron olmayan veriler için standart LINQ yöntemlerini kullanın.

```csharp
var inMemoryList = new List<int> { 1, 2, 3 };
var result = inMemoryList.ToList();
```

---

## 5. `FirstAsync` ve `SingleAsync` Kullanımını Yanlış Yönetmek

❌ **Yanlış Kullanım:** Çok fazla sonuç döndüren bir sorgu için `SingleAsync` kullanmak.

```csharp
var user = await context.Users.SingleAsync(u => u.IsActive); // Birden fazla sonuç dönerse hata
```

✅ **İdeal Kullanım:** Sonuçların birden fazla olabileceği durumlarda `FirstOrDefaultAsync` kullanın.

```csharp
var user = await context.Users.FirstOrDefaultAsync(u => u.IsActive);
if (user == null)
{
    Console.WriteLine("Kullanıcı bulunamadı.");
}
```

---

## 6. `AsNoTracking` Kullanımını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Sorgu sonuçlarını yalnızca okuma amaçlı kullanırken izleme (tracking) yapmayı ihmal etmek.

```csharp
var products = await context.Products.ToListAsync(); // Tracking açık
```

✅ **İdeal Kullanım:** Yalnızca okuma amaçlı sorgularda `AsNoTracking` kullanın.

```csharp
var products = await context.Products.AsNoTracking().ToListAsync();
```