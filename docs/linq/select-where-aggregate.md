# Select, Where ve Aggregate

LINQ sorguları, koleksiyonlar ve veritabanı işlemleri üzerinde etkili bir şekilde veri işlemek için güçlü bir araçtır. Ancak, `Select`, `Where` ve `Aggregate` gibi LINQ yöntemlerinin yanlış kullanımı performans kaybına ve karmaşık kod yapısına yol açabilir.

---

## 1. Gereksiz `Select` Kullanımı

❌ **Yanlış Kullanım:** Gerekli olmayan veri projeksiyonları yapmak.

```csharp
var productNames = context.Products
    .Select(p => new { p.Name, p.Price })
    .Select(p => p.Name)
    .ToList();
```

✅ **İdeal Kullanım:** Gereksiz `Select` projeksiyonlarından kaçının.

```csharp
var productNames = context.Products
    .Select(p => p.Name)
    .ToList();
```

---

## 2. `Where` ile Karmaşık Filtreler Yazmak

❌ **Yanlış Kullanım:** `Where` içinde karmaşık mantık kullanarak okunabilirliği azaltmak.

```csharp
var expensiveProducts = context.Products
    .Where(p => p.Price > 100 && p.Stock > 0 && p.Category == "Electronics")
    .ToList();
```

✅ **İdeal Kullanım:** Filtreleri yardımcı metotlarla bölerek okunabilirliği artırın.

```csharp
var expensiveProducts = context.Products
    .Where(IsExpensiveAndInStock)
    .ToList();

bool IsExpensiveAndInStock(Product product) =>
    product.Price > 100 && product.Stock > 0 && product.Category == "Electronics";
```

---

## 3. `Aggregate`'i Yanlış Kullanmak

❌ **Yanlış Kullanım:** Basit toplam veya birleştirme işlemleri için `Aggregate` kullanmak.

```csharp
var totalStock = context.Products
    .Select(p => p.Stock)
    .Aggregate(0, (acc, stock) => acc + stock);
```

✅ **İdeal Kullanım:** Basit işlemler için uygun LINQ yöntemlerini kullanın.

```csharp
var totalStock = context.Products.Sum(p => p.Stock);
```

---

## 4. Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Filtreleri veritabanı yerine bellek içinde yapmak.

```csharp
var products = context.Products.ToList();
var expensiveProducts = products.Where(p => p.Price > 100);
```

✅ **İdeal Kullanım:** Filtreleme işlemlerini veritabanı tarafında yapın.

```csharp
var expensiveProducts = context.Products
    .Where(p => p.Price > 100)
    .ToList();
```

---

## 5. Çok Aşamalı Sorguların Karmaşıklaştırılması

❌ **Yanlış Kullanım:** Birden fazla LINQ zinciri ile karmaşık sorgular yazmak.

```csharp
var productData = context.Products
    .Where(p => p.Price > 100)
    .Select(p => new { p.Name, p.Stock })
    .Where(p => p.Stock > 10)
    .ToList();
```

✅ **İdeal Kullanım:** Sorguları açık ve tek bir zincir halinde yazın.

```csharp
var productData = context.Products
    .Where(p => p.Price > 100 && p.Stock > 10)
    .Select(p => new { p.Name, p.Stock })
    .ToList();
```

---

## 6. Hatalı `First` ve `Single` Kullanımı

❌ **Yanlış Kullanım:** `First` veya `Single` kullanarak hata riski oluşturmak.

```csharp
var product = context.Products.Single(p => p.Id == 1); // Veri yoksa hata verir.
```

✅ **İdeal Kullanım:** `FirstOrDefault` veya `SingleOrDefault` kullanarak güvenli sorgulama yapın.

```csharp
var product = context.Products.SingleOrDefault(p => p.Id == 1);
if (product == null)
{
    Console.WriteLine("Ürün bulunamadı.");
}
```