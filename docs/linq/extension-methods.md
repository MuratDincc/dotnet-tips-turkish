# LINQ Extension Methods

LINQ (Language Integrated Query), C# ile veri işlemlerini kolaylaştırmak için güçlü bir araçtır. LINQ, genişletme metotları (extension methods) ile sorgu yazımını daha esnek ve okunabilir hale getirir. Ancak, bu metotların yanlış kullanımı performans kaybına ve karmaşık kod yapılarına yol açabilir.

---

## 1. Gereksiz `ToList` Kullanımı

❌ **Yanlış Kullanım:** Sorguların her adımında `ToList` kullanmak.

```csharp
var filteredData = context.Data
    .Where(d => d.IsActive)
    .ToList()
    .Select(d => d.Name)
    .ToList();
```

✅ **İdeal Kullanım:** Sorguyu tek bir işlemde yürütün.

```csharp
var filteredData = context.Data
    .Where(d => d.IsActive)
    .Select(d => d.Name)
    .ToList();
```

---

## 2. Büyük Veri Setlerinde `OrderBy` ile Performans Sorunları

❌ **Yanlış Kullanım:** Bellekte sıralama yapmak.

```csharp
var data = context.Data.ToList().OrderBy(d => d.Name).ToList();
```

✅ **İdeal Kullanım:** Sıralama işlemini veritabanında gerçekleştirin.

```csharp
var data = context.Data
    .OrderBy(d => d.Name)
    .ToList();
```

---

## 3. Gereksiz `Select` Kullanımı

❌ **Yanlış Kullanım:** Gereksiz projeksiyonlar yapmak.

```csharp
var data = context.Data
    .Select(d => new { d.Id, d.Name })
    .Select(d => d.Name)
    .ToList();
```

✅ **İdeal Kullanım:** Doğrudan ihtiyacınız olan veriyi seçin.

```csharp
var data = context.Data
    .Select(d => d.Name)
    .ToList();
```

---

## 4. `First` ve `Single` Kullanımını Yanlış Yönetmek

❌ **Yanlış Kullanım:** Veri bulunamaması durumunda hata veren `First` veya `Single` kullanmak.

```csharp
var item = context.Data.First(d => d.Id == 1); // Veri yoksa hata fırlatır
```

✅ **İdeal Kullanım:** Güvenli sorgulamalar için `FirstOrDefault` veya `SingleOrDefault` kullanın.

```csharp
var item = context.Data.FirstOrDefault(d => d.Id == 1);
if (item == null)
{
    Console.WriteLine("Veri bulunamadı.");
}
```

---

## 5. `Count` Kullanımıyla Performansı Etkilemek

❌ **Yanlış Kullanım:** `Count`'u bellek içindeki bir koleksiyona uygulamak.

```csharp
var count = context.Data.ToList().Count;
```

✅ **İdeal Kullanım:** Veritabanında `Count` işlemini gerçekleştirin.

```csharp
var count = context.Data.Count();
```

---

## 6. Genişletme Metotları ile Karmaşık Yapılar Yazmak

❌ **Yanlış Kullanım:** Tek satırda karmaşık işlemleri zincirlemek.

```csharp
var data = context.Data
    .Where(d => d.IsActive)
    .OrderBy(d => d.Name)
    .Select(d => new { d.Id, d.Name, d.Date })
    .ToList()
    .GroupBy(d => d.Date.Year);
```

✅ **İdeal Kullanım:** İşlemleri adımlara bölerek kodu daha okunabilir hale getirin.

```csharp
var activeData = context.Data
    .Where(d => d.IsActive)
    .OrderBy(d => d.Name)
    .Select(d => new { d.Id, d.Name, d.Date })
    .ToList();

var groupedData = activeData.GroupBy(d => d.Date.Year);
```

---

## 7. `Any` ve `Exists` Kullanımını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Var olup olmadığını kontrol etmek için `Count` kullanmak.

```csharp
var exists = context.Data.Count(d => d.IsActive) > 0;
```

✅ **İdeal Kullanım:** Daha performanslı `Any` metodunu kullanın.

```csharp
var exists = context.Data.Any(d => d.IsActive);
```