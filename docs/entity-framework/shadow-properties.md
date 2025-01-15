# Shadow Properties

Shadow properties, Entity Framework Core'da bir entity üzerinde tanımlanmayan ancak veritabanında saklanan özelliklerdir. Bu özellikler, özellikle genişletilebilirlik ve esneklik sağlasa da, yanlış kullanımları veri tutarsızlıklarına ve kod karmaşıklığına yol açabilir.

---

## 1. Shadow Properties'i Gereksiz Kullanmak

❌ **Yanlış Kullanım:** Entity sınıfında tanımlanabilecek bir özelliği shadow property olarak kullanmak.

```csharp
modelBuilder.Entity<Product>()
    .Property<DateTime>("LastUpdated");
```

✅ **İdeal Kullanım:** Gereksiz shadow property kullanımından kaçının ve entity sınıfına ekleyin.

```csharp
public class Product
{
    public int Id { get; set; }
    public DateTime LastUpdated { get; set; }
}
```

---

## 2. Shadow Properties'in Değerlerini Yanlış Yönetmek

❌ **Yanlış Kullanım:** Shadow property değerini kontrol etmeden kullanmak.

```csharp
var lastUpdated = context.Entry(product).Property("LastUpdated").CurrentValue;
Console.WriteLine(lastUpdated);
```

✅ **İdeal Kullanım:** Shadow property değerlerini güvenli bir şekilde yönetin.

```csharp
var lastUpdatedProperty = context.Entry(product).Property("LastUpdated");
if (lastUpdatedProperty != null)
{
    Console.WriteLine(lastUpdatedProperty.CurrentValue);
}
else
{
    Console.WriteLine("LastUpdated özelliği mevcut değil.");
}
```

---

## 3. Veri Tutarlılığını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Shadow property değerlerini güncellemeden bırakmak.

```csharp
var product = context.Products.Find(1);
context.Entry(product).Property("LastUpdated").CurrentValue = null; // Değer tutarsızlığı yaratır
context.SaveChanges();
```

✅ **İdeal Kullanım:** Shadow property değerlerini her işlemde uygun şekilde güncelleyin.

```csharp
var product = context.Products.Find(1);
context.Entry(product).Property("LastUpdated").CurrentValue = DateTime.UtcNow;
context.SaveChanges();
```

---

## 4. Shadow Properties'i Hata Ayıklama Sürecinde Göz Ardı Etmek

❌ **Yanlış Kullanım:** Shadow property'lerin hata ayıklama sırasında görünürlüğünü sağlamamak.

```csharp
var product = context.Products.Find(1);
// Shadow property değerlerini incelemeden geçmek
```

✅ **İdeal Kullanım:** Hata ayıklama sırasında shadow property'lerin değerlerini kontrol edin.

```csharp
var product = context.Products.Find(1);
var lastUpdated = context.Entry(product).Property("LastUpdated").CurrentValue;
Console.WriteLine($"LastUpdated: {lastUpdated}");
```

---

## 5. Shadow Properties ile Yanlış İlişkiler Kurmak

❌ **Yanlış Kullanım:** Shadow property'leri ilişkilerde doğrudan kullanmak.

```csharp
modelBuilder.Entity<Order>()
    .HasOne<Product>()
    .WithMany()
    .HasForeignKey("ProductId");
```

✅ **İdeal Kullanım:** Shadow property yerine açıkça tanımlanmış ilişkiler kullanın.

```csharp
public class Order
{
    public int Id { get; set; }
    public int ProductId { get; set; }
    public Product Product { get; set; }
}
```