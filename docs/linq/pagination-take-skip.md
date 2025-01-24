# LINQ ile Sayfalama: Take ve Skip Kullanımı

Büyük veri kümeleriyle çalışırken verilerin belirli bir kısmını almak veya sayfa bazlı veri döndürmek yaygın bir ihtiyaçtır. LINQ `Take` ve `Skip` metotları, bu ihtiyacı karşılamak için kullanılır.

---

## 1. Take ve Skip Nedir?

- **Take:** Verilen sayıda elemanı alır.
- **Skip:** Verilen sayıda elemanı atlar.

Bu iki metot bir arada kullanıldığında sayfalama işlemleri kolayca gerçekleştirilebilir.

---

## 2. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** Tüm veriyi getirmek

❌ **Yanlış Kullanım:**

```csharp
var allData = data.ToList(); // Tüm veriyi bellek içine alır
var pageData = allData.Skip(10).Take(10).ToList(); // Gereksiz bellek tüketimi
```

Bu yöntem, gereksiz yere tüm veriyi belleğe yükler ve performans kaybına yol açar.

---

### **İdeal Kullanım:** Doğrudan sorguda Take ve Skip kullanımı

✅ **İdeal Kullanım:**

```csharp
var pageData = data.Skip(10).Take(10).ToList(); // Sorgu veritabanında çalışır
```

Bu yöntem, veritabanında yalnızca ihtiyaç duyulan verinin alınmasını sağlar.

---

## 3. Sayfalama Örneği

Aşağıdaki örnek, bir koleksiyon üzerinde sayfalama yapmanın temel mantığını gösterir:

```csharp
int pageNumber = 2;
int pageSize = 10;

var pageData = data
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToList();

foreach (var item in pageData)
{
    Console.WriteLine(item);
}
```

Bu kod, 2. sayfadan 10 eleman alır.

---

## 4. Performans İyileştirme

Veri tabanında büyük koleksiyonlarla çalışırken `AsQueryable` kullanarak performans artışı sağlayabilirsiniz:

✅ **Örnek:**

```csharp
var pageData = context.Customers
    .AsQueryable()
    .Skip(20)
    .Take(10)
    .ToList();
```

Bu sorgu, yalnızca ihtiyaç duyulan veriyi veritabanından çeker ve belleği optimize eder.

---

## 5. Dinamik Sayfalama

Kullanıcı girişine göre dinamik olarak sayfalama işlemi yapılabilir:

✅ **Örnek:**

```csharp
public List<T> GetPagedData<T>(IQueryable<T> query, int pageNumber, int pageSize)
{
    return query
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToList();
}

// Kullanım
var customersPage = GetPagedData(context.Customers, 3, 15);
```

---

## 6. Toplam Sayfa Sayısı Hesaplama

Sayfalama yaparken toplam sayfa sayısını hesaplamak için veri setindeki toplam eleman sayısı kullanılabilir:

✅ **Örnek:**

```csharp
int totalItems = context.Customers.Count();
int pageSize = 10;
int totalPages = (int)Math.Ceiling((double)totalItems / pageSize);

Console.WriteLine($"Toplam Sayfa: {totalPages}");
```

---

## 7. İleri ve Geri Gezinme

Kullanıcıların sayfalar arasında kolayca gezinebilmeleri için ileri ve geri gezinme mantığı uygulanabilir:

✅ **Örnek:**

```csharp
int currentPage = 1;

var nextPageData = data
    .Skip(currentPage * pageSize)
    .Take(pageSize)
    .ToList();

currentPage--;

var previousPageData = data
    .Skip((currentPage - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```