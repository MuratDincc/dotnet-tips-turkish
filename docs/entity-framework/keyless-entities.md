# Keyless Entities

Entity Framework Core'da "Keyless Entities", birincil anahtara ihtiyaç duymayan ve genellikle yalnızca okuma amaçlı kullanılan varlıkları temsil eder. Bu özellik, görünümler, birincil anahtarsız tablolar veya özel SQL sorgularını haritalamak gibi durumlarda oldukça faydalıdır. Ancak, yanlış kullanımı performans ve veri tutarlılığı sorunlarına neden olabilir.

---

## 1. Keyless Entities Kullanımını Yanlış Anlamak

❌ **Yanlış Kullanım:** Keyless entities'i değişiklik takibi (change tracking) için kullanmak.

```csharp
[Keyless]
public class Report
{
    public int Id { get; set; } // Yanlış: Keyless entity'de birincil anahtar olmamalıdır.
    public string ReportName { get; set; }
}
```

✅ **İdeal Kullanım:** Keyless entities yalnızca okuma amaçlı kullanılmalıdır.

```csharp
[Keyless]
public class Report
{
    public string ReportName { get; set; }
    public DateTime GeneratedOn { get; set; }
}
```

---

## 2. Gereksiz Değişiklik Takibi Yapmak

❌ **Yanlış Kullanım:** Keyless entities ile veri güncellemeye çalışmak.

```csharp
var report = new Report { ReportName = "Annual Report" };
context.Reports.Add(report); // Hata: Keyless entities eklenemez
context.SaveChanges();
```

✅ **İdeal Kullanım:** Keyless entities yalnızca sorgulama amacıyla kullanılmalıdır.

```csharp
var reports = context.Reports
    .Where(r => r.GeneratedOn > DateTime.UtcNow.AddDays(-30))
    .ToList();
```

---

## 3. Keyless Entities'i Varsayılan Şekilde Kullanmak

❌ **Yanlış Kullanım:** Keyless entities'i tanımlarken gerekli yapılandırmaları yapmamak.

```csharp
public class Report
{
    public string ReportName { get; set; }
    public DateTime GeneratedOn { get; set; }
}
```

✅ **İdeal Kullanım:** `[Keyless]` veya `.HasNoKey()` ile açıkça yapılandırma yapılmalıdır.

```csharp
[Keyless]
public class Report
{
    public string ReportName { get; set; }
    public DateTime GeneratedOn { get; set; }
}
```

Veya Fluent API kullanarak:

```csharp
modelBuilder.Entity<Report>().HasNoKey();
```

---

## 4. Performans Optimizasyonunu Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri setlerini sorgularken keyless entities için optimize edilmemiş sorgular yazmak.

```csharp
var reports = context.Reports.ToList(); // Tüm veri setini yükler
```

✅ **İdeal Kullanım:** Sorguları filtreleyerek optimize edin.

```csharp
var recentReports = context.Reports
    .Where(r => r.GeneratedOn > DateTime.UtcNow.AddMonths(-1))
    .ToList();
```

---

## 5. Veri Tutarlılığını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Keyless entities'i ilişkisel verilerle hatalı bir şekilde kullanmak.

```csharp
public class OrderSummary
{
    public int OrderId { get; set; }
    public decimal Total { get; set; }
    public string CustomerName { get; set; }
}
```

✅ **İdeal Kullanım:** Keyless entities'deki veriler yalnızca okunabilir olmalıdır.

```csharp
[Keyless]
public class OrderSummary
{
    public decimal Total { get; set; }
    public string CustomerName { get; set; }
}
```