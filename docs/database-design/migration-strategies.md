# Migration Stratejileri ve Connection Pooling

Veritabanı migration yönetimi ve connection pooling, uygulamanın kararlılığı ve performansı için kritiktir; yanlış yönetim veri kaybına ve bağlantı sorunlarına yol açar.

---

## 1. Migration'ları Uygulama Başlangıcında Çalıştırmak

❌ **Yanlış Kullanım:** `Database.Migrate()` ile otomatik migration.

```csharp
var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate(); // Production'da riskli, birden fazla instance'da race condition
}
```

✅ **İdeal Kullanım:** Migration'ları CI/CD pipeline'ında ayrı adım olarak çalıştırın.

```csharp
// Program.cs - Sadece development'ta
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}

// Production: CI/CD pipeline'ında
// dotnet ef database update --connection "ProductionConnectionString"

// Veya Bundle ile
// dotnet ef migrations bundle --self-contained -o efbundle
// ./efbundle --connection "ProductionConnectionString"
```

---

## 2. Destructive Migration Yapmak

❌ **Yanlış Kullanım:** Kolon silme veya yeniden adlandırma ile veri kaybı.

```csharp
// Migration: FullName kolonu eklendi, Name ve Surname silindi
migrationBuilder.DropColumn(name: "Name", table: "Users");
migrationBuilder.DropColumn(name: "Surname", table: "Users");
migrationBuilder.AddColumn<string>(name: "FullName", table: "Users");
// Name ve Surname verileri kaybolur!
```

✅ **İdeal Kullanım:** Güvenli migration pattern ile veri kaybını önleyin.

```csharp
// Adım 1: Yeni kolon ekle ve veriyi kopyala
migrationBuilder.AddColumn<string>(name: "FullName", table: "Users", nullable: true);
migrationBuilder.Sql("UPDATE Users SET FullName = Name + ' ' + Surname");

// Adım 2: Uygulama hem eski hem yeni kolonu kullanır (geçiş dönemi)

// Adım 3: Yeni kolon zorunlu yapılıp eski kolonlar kaldırılır (ayrı migration)
migrationBuilder.AlterColumn<string>(name: "FullName", table: "Users", nullable: false);
migrationBuilder.DropColumn(name: "Name", table: "Users");
migrationBuilder.DropColumn(name: "Surname", table: "Users");
```

---

## 3. Connection String'i Hardcode Etmek

❌ **Yanlış Kullanım:** Bağlantı bilgilerini koda gömmek.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("Server=prod-server;Database=MyApp;User=sa;Password=P@ssw0rd!"));
// Şifre kaynak kodda, güvenlik riski
```

✅ **İdeal Kullanım:** Ortam bazlı konfigürasyon ve secret yönetimi kullanın.

```csharp
// appsettings.json (development)
// { "ConnectionStrings": { "Default": "Server=localhost;..." } }

// Production: Environment variable veya Azure Key Vault
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("Default"),
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(10),
                errorNumbersToAdd: null);
            sqlOptions.CommandTimeout(30);
        }));

// Secret Manager (development)
// dotnet user-secrets set "ConnectionStrings:Default" "Server=..."
```

---

## 4. Connection Pool Ayarlarını İhmal Etmek

❌ **Yanlış Kullanım:** Varsayılan pool ayarları ile bağlantı tükenmesi.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("Server=..."));
// Varsayılan max pool size: 100, yoğun trafikte yetersiz kalabilir
// Bağlantılar açık kalırsa pool tükenir
```

✅ **İdeal Kullanım:** Connection pool ayarlarını optimize edin.

```csharp
// Connection string'de pool ayarları
var connectionString = "Server=...;Max Pool Size=200;Min Pool Size=10;" +
                       "Connection Lifetime=300;Pooling=true";

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// DbContext Pooling (daha performanslı)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseSqlServer(connectionString),
    poolSize: 128);

// Health check ile pool durumunu izleme
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database",
        timeout: TimeSpan.FromSeconds(3));
```

---

## 5. Concurrency Kontrolü Yapmamak

❌ **Yanlış Kullanım:** Eşzamanlı güncellemelerde veri kaybı.

```csharp
var product = await _context.Products.FindAsync(id);
product.Stock -= quantity;
await _context.SaveChangesAsync();
// İki kullanıcı aynı anda satın alırsa stok yanlış azalır
```

✅ **İdeal Kullanım:** Optimistic concurrency ile veri tutarlılığını sağlayın.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Stock { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } // Concurrency token
}

public async Task<Result> ReduceStockAsync(int productId, int quantity)
{
    var product = await _context.Products.FindAsync(productId);

    if (product.Stock < quantity)
        return Result.Failure("Yetersiz stok");

    product.Stock -= quantity;

    try
    {
        await _context.SaveChangesAsync();
        return Result.Success();
    }
    catch (DbUpdateConcurrencyException)
    {
        // Başka biri güncellemiş, yeniden dene
        _context.Entry(product).State = EntityState.Detached;
        return await ReduceStockAsync(productId, quantity);
    }
}
```
