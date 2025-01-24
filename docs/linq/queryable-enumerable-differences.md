# LINQ: IQueryable ve IEnumerable Arasındaki Farklar

LINQ ile çalışırken `IQueryable` ve `IEnumerable` arayüzleri arasında doğru seçim yapmak, performans ve sorgu davranışı açısından kritik öneme sahiptir. Bu iki arayüzün işleyişi ve kullanım alanları farklıdır.

---

## 1. IQueryable Nedir?

- **Tanım:** Sorgunun veritabanı veya uzak bir kaynaktan yürütülmesine olanak tanır.
- **Çalışma Prensibi:** Sorgular, veritabanına gönderilir ve işlenir (deferred execution).

**Örnek:**

```csharp
using var context = new AppDbContext();

IQueryable<Customer> query = context.Customers.Where(c => c.IsActive);
var activeCustomers = query.ToList(); // Sorgu burada çalışır
```

`IQueryable` sayesinde sorgu, veritabanında çalıştırılır ve sadece ihtiyaç duyulan veriler çekilir.

---

## 2. IEnumerable Nedir?

- **Tanım:** Veriyi belleğe yükler ve bellekte işleme alır.
- **Çalışma Prensibi:** Sorgular, bellekte yürütülür.

**Örnek:**

```csharp
IEnumerable<Customer> customers = context.Customers.ToList();
var activeCustomers = customers.Where(c => c.IsActive).ToList(); // Filtreleme bellekte yapılır
```

`IEnumerable`, tüm veriyi belleğe çeker ve filtreleme işlemi bellek üzerinde yapılır.

---

## 3. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** Büyük veri kümelerini belleğe çekmek

❌ **Yanlış Kullanım:**

```csharp
var allCustomers = context.Customers.ToList(); // Tüm veriyi belleğe çeker
var activeCustomers = allCustomers.Where(c => c.IsActive).ToList();
```

Bu yaklaşım, büyük veri kümeleri için gereksiz bellek tüketimine yol açar.

---

### **İdeal Kullanım:** Sorguları veritabanında çalıştırmak

✅ **İdeal Kullanım:**

```csharp
var activeCustomers = context.Customers
    .Where(c => c.IsActive)
    .ToList(); // Sorgu doğrudan veritabanında çalışır
```

Bu yöntem, yalnızca gerekli veriyi çekerek bellek tüketimini optimize eder.

---

## 4. IQueryable ve IEnumerable Farkları

| Özellik                     | IQueryable                               | IEnumerable                             |
|-----------------------------|------------------------------------------|-----------------------------------------|
| **Çalışma Yeri**             | Veritabanı veya uzak kaynak              | Bellek                                  |
| **Performans**               | Daha iyi (sorgu kaynağında çalıştırılır) | Daha düşük (veri bellekte işlenir)      |
| **Kullanım Alanı**           | Büyük veri kümeleri, veritabanı sorguları | Küçük veri kümeleri, bellek işlemleri   |
| **Lazy Execution**           | Evet                                    | Evet                                    |
| **Sorgu İşleme**             | SQL gibi sorgu dilleri                  | Bellek üstünde LINQ                     |

---

## 5. Dinamik Sorgu Örneği

✅ **Örnek:** Kullanıcı girişine bağlı sorgu

```csharp
public List<Customer> GetCustomers(bool onlyActive)
{
    using var context = new AppDbContext();

    IQueryable<Customer> query = context.Customers;

    if (onlyActive)
    {
        query = query.Where(c => c.IsActive);
    }

    return query.ToList();
}
```

Bu yöntem, sorguları yalnızca ihtiyaç duyulan verilere göre dinamik olarak optimize eder.

---

## 6. Performans Testi

`IQueryable` ve `IEnumerable` arasındaki performans farkını test etmek için aşağıdaki kodu kullanabilirsiniz:

✅ **Performans Testi:**

```csharp
var stopwatch = Stopwatch.StartNew();

// IQueryable ile
var activeCustomersQuery = context.Customers.Where(c => c.IsActive).ToList();
stopwatch.Stop();
Console.WriteLine($"IQueryable Süresi: {stopwatch.ElapsedMilliseconds} ms");

stopwatch.Restart();

// IEnumerable ile
var allCustomers = context.Customers.ToList();
var activeCustomersEnum = allCustomers.Where(c => c.IsActive).ToList();
stopwatch.Stop();
Console.WriteLine($"IEnumerable Süresi: {stopwatch.ElapsedMilliseconds} ms");
```

---

## 7. Hangi Durumda Hangisi Kullanılmalı?

| **Durum**                                | **Tercih Edilen Arayüz** |
|------------------------------------------|--------------------------|
| Veritabanı işlemleri                     | IQueryable               |
| Bellekte küçük veri kümeleriyle çalışma  | IEnumerable              |
| Dinamik sorgu oluşturma                  | IQueryable               |
| Performans kritik olan büyük veri setleri | IQueryable               |