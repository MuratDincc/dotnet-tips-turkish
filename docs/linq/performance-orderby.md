# LINQ ile Performanslı Sıralama: OrderBy Kullanımı

Veri sıralama işlemleri, genellikle büyük veri kümelerinde performansı etkileyen önemli bir adımdır. LINQ `OrderBy` ve `ThenBy` metotları ile sıralama işlemleri gerçekleştirilir.

---

## 1. OrderBy ve ThenBy Nedir?

- **OrderBy:** Veriyi belirli bir sütuna göre artan sırada sıralar.
- **ThenBy:** Önceki sıralama kriterinden sonra ikinci bir sıralama uygular.

**Örnek Kullanım:**

```csharp
var sortedData = data.OrderBy(x => x.Name).ThenBy(x => x.Age).ToList();
```

Bu kod, `Name` alanına göre artan sıralama yapar. Eğer `Name` aynıysa, `Age` alanına göre sıralama yapar.

---

## 2. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** Çok sayıda sıralama işlemi

❌ **Yanlış Kullanım:**

```csharp
var sortedData = data
    .OrderBy(x => x.Name)
    .OrderBy(x => x.Age)
    .ToList();
```

Bu kod, her `OrderBy` çağrısında sıralama işlemini yeniden başlatır. Bu nedenle performans kaybına neden olur.

---

### **İdeal Kullanım:** ThenBy ile sıralamaları birleştirme

✅ **İdeal Kullanım:**

```csharp
var sortedData = data
    .OrderBy(x => x.Name)
    .ThenBy(x => x.Age)
    .ToList();
```

Bu yöntem, sıralamaları birleştirerek daha verimli bir işlem yapar.

---

## 3. Sıralama Performansını Artırmak

### **1. AsQueryable Kullanımı**

Veri tabanı sorgularında, sıralama işlemini belleğe taşımadan önce veritabanında gerçekleştirmek daha performanslıdır.

✅ **Örnek:**

```csharp
var sortedData = context.Customers
    .AsQueryable()
    .OrderBy(x => x.Name)
    .ToList();
```

### **2. Sütun İndeksleri**

Veritabanında sıralama yapılan sütunlarda indeks oluşturmak, sıralama performansını ciddi şekilde artırır.

```sql
CREATE INDEX idx_name ON Customers (Name);
```

### **3. Azalan Sıralama (Descending Order)**

LINQ `OrderByDescending` ile veriyi azalan sırada sıralayabilirsiniz.

✅ **Örnek:**

```csharp
var sortedData = data.OrderByDescending(x => x.Date).ToList();
```

---

## 4. Dinamik Sıralama

Kullanıcıdan gelen girişlere bağlı olarak sıralama yapmak gerekebilir.

✅ **Örnek:**

```csharp
public List<T> SortData<T>(IQueryable<T> query, string sortColumn, bool ascending)
{
    var parameter = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(parameter, sortColumn);
    var lambda = Expression.Lambda<Func<T, object>>(Expression.Convert(property, typeof(object)), parameter);

    return ascending
        ? query.OrderBy(lambda).ToList()
        : query.OrderByDescending(lambda).ToList();
}

// Kullanım
var sortedCustomers = SortData(context.Customers, "Name", true);
```

---

## 5. Birden Fazla Alan ile Sıralama

Birden fazla alanı sıralama kriteri olarak belirlemek için `ThenBy` ve `ThenByDescending` kullanılabilir.

✅ **Örnek:**

```csharp
var sortedData = data
    .OrderBy(x => x.LastName)
    .ThenByDescending(x => x.FirstName)
    .ToList();
```

Bu kod, önce `LastName` alanına göre sıralama yapar, aynı değerlerde ise `FirstName` alanına göre azalan sıralama yapar.

---

## 6. Performans Testi

Sıralama işlemlerinin performansını ölçmek için `Stopwatch` kullanabilirsiniz.

✅ **Örnek:**

```csharp
var stopwatch = Stopwatch.StartNew();

var sortedData = data.OrderBy(x => x.Name).ToList();

stopwatch.Stop();
Console.WriteLine($"Sıralama süresi: {stopwatch.ElapsedMilliseconds} ms");
```