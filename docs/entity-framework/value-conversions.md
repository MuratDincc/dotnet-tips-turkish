# Value Conversions

Entity Framework Core'da Value Conversions, bir entity property'sinin veritabanında farklı bir formatta saklanmasını sağlar. Bu özellik, özel veri türlerini desteklemek ve esneklik sağlamak için oldukça kullanışlıdır. Ancak, yanlış kullanımı veri kayıplarına ve performans sorunlarına neden olabilir.

---

## 1. Gereksiz Value Converter Kullanımı

❌ **Yanlış Kullanım:** Basit türler için gereksiz Value Converter tanımlamak.

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .HasConversion(
        v => v.ToString(),
        v => decimal.Parse(v));
```

✅ **İdeal Kullanım:** Value Converter'ı yalnızca gerektiğinde tanımlayın.

```csharp
public class Product
{
    public decimal Price { get; set; }
}
```

---

## 2. Veri Kayıplarını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Dönüşüm sırasında veri kaybını göz ardı etmek.

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Rating)
    .HasConversion(
        v => (int)v,  // Veri kaybı riski
        v => (double)v);
```

✅ **İdeal Kullanım:** Dönüşümün veri bütünlüğünü koruyacak şekilde yapılması.

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Rating)
    .HasConversion(
        v => Math.Round(v, 2),  // Hassasiyet korunur
        v => v);
```

---

## 3. Karmaşık Dönüşümleri Property Düzeyinde Yapmak

❌ **Yanlış Kullanım:** Karmaşık dönüşümleri Value Converter içinde yapmak.

```csharp
modelBuilder.Entity<User>()
    .Property(u => u.Roles)
    .HasConversion(
        v => string.Join(",", v),  // Karmaşık dönüşüm
        v => v.Split(','));
```

✅ **İdeal Kullanım:** Karmaşık dönüşümleri ayrı bir sınıf veya metotla yönetmek.

```csharp
public class RoleConverter : ValueConverter<List<string>, string>
{
    public RoleConverter()
        : base(
            v => string.Join(",", v),
            v => v.Split(',').ToList())
    {
    }
}

modelBuilder.Entity<User>()
    .Property(u => u.Roles)
    .HasConversion(new RoleConverter());
```

---

## 4. Tarih ve Saat Dönüşümlerinde Yanlış Format Kullanımı

❌ **Yanlış Kullanım:** Tarih ve saat dönüşümlerinde standart formatı kullanmamak.

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.OrderDate)
    .HasConversion(
        v => v.ToString(),
        v => DateTime.Parse(v)); // Kültür farkları sorun yaratabilir
```

✅ **İdeal Kullanım:** Tarih ve saat dönüşümlerinde `DateTimeOffset` kullanmak.

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.OrderDate)
    .HasConversion(
        v => v.ToString("o"),
        v => DateTimeOffset.Parse(v));
```

---

## 5. Performans Maliyetlerini Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri kümelerinde performans maliyetini göz ardı etmek.

```csharp
modelBuilder.Entity<Log>()
    .Property(l => l.Details)
    .HasConversion(
        v => JsonConvert.SerializeObject(v),
        v => JsonConvert.DeserializeObject<Dictionary<string, string>>(v)); // Yoğun JSON dönüşümü
```

✅ **İdeal Kullanım:** Performansı optimize eden daha hızlı dönüşümler kullanmak.

```csharp
modelBuilder.Entity<Log>()
    .Property(l => l.Details)
    .HasConversion(
        v => System.Text.Json.JsonSerializer.Serialize(v),
        v => System.Text.Json.JsonSerializer.Deserialize<Dictionary<string, string>>(v));
```

---

## 6. Test Edilebilirliği Göz Ardı Etmek

❌ **Yanlış Kullanım:** Dönüşüm mantığını test edilebilir hale getirmemek.

```csharp
modelBuilder.Entity<User>()
    .Property(u => u.Preferences)
    .HasConversion(
        v => string.Join(";", v),
        v => v.Split(';').ToList());
```

✅ **İdeal Kullanım:** Dönüşüm mantığını test edilebilir hale getirmek için ayrı bir sınıf kullanmak.

```csharp
public class PreferencesConverter : ValueConverter<List<string>, string>
{
    public PreferencesConverter()
        : base(
            v => string.Join(";", v),
            v => v.Split(';').ToList())
    {
    }
}

modelBuilder.Entity<User>()
    .Property(u => u.Preferences)
    .HasConversion(new PreferencesConverter());
```