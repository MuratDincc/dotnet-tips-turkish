# Indexing ve Query Optimization

Doğru indexleme ve sorgu optimizasyonu veritabanı performansının temelidir; eksik veya yanlış index'ler yavaş sorgulara ve kaynak israfına yol açar.

---

## 1. Index Tanımlamamak

❌ **Yanlış Kullanım:** Sık sorgulanan alanlarda index olmadan çalışmak.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
}

// Her sorguda full table scan
var orders = await _context.Orders
    .Where(o => o.CustomerId == customerId && o.Status == "Active")
    .ToListAsync();
```

✅ **İdeal Kullanım:** Sorgu pattern'lerine göre index tanımlayın.

```csharp
public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>(entity =>
        {
            // Tek alan index
            entity.HasIndex(o => o.CustomerId);

            // Composite index (sık kullanılan filtre kombinasyonu)
            entity.HasIndex(o => new { o.CustomerId, o.Status })
                .HasDatabaseName("IX_Orders_CustomerId_Status");

            // Unique index
            entity.HasIndex(o => o.OrderNumber).IsUnique();

            // Filtered index (sadece aktif kayıtlar)
            entity.HasIndex(o => o.CreatedAt)
                .HasFilter("[Status] = 'Active'")
                .HasDatabaseName("IX_Orders_CreatedAt_ActiveOnly");
        });
    }
}
```

---

## 2. Select * ile Tüm Alanları Çekmek

❌ **Yanlış Kullanım:** İhtiyaç dışı alanları sorgulamak.

```csharp
var orders = await _context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync();
// Tüm navigation property'ler ve alanlar çekilir
```

✅ **İdeal Kullanım:** Projection ile sadece gerekli alanları seçin.

```csharp
var orders = await _context.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        OrderNumber = o.OrderNumber,
        TotalAmount = o.OrderItems.Sum(oi => oi.Price * oi.Quantity),
        CustomerName = o.Customer.Name,
        ItemCount = o.OrderItems.Count,
        CreatedAt = o.CreatedAt
    })
    .ToListAsync();
// Tek sorgu, sadece gerekli alanlar, JOIN otomatik
```

---

## 3. Tracking'i Gereksiz Yere Açık Bırakmak

❌ **Yanlış Kullanım:** Read-only sorgularda change tracking aktif.

```csharp
var products = await _context.Products
    .Where(p => p.IsActive)
    .ToListAsync(); // Change tracker tüm entity'leri izler
// Sadece okuma yapılacak ama bellek ve CPU harcanır
```

✅ **İdeal Kullanım:** Read-only sorgularda AsNoTracking kullanın.

```csharp
// Tek sorguda
var products = await _context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// DbContext seviyesinde (read-only context)
public class ReadOnlyDbContext : DbContext
{
    public ReadOnlyDbContext(DbContextOptions<ReadOnlyDbContext> options)
        : base(options)
    {
        ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
        ChangeTracker.AutoDetectChangesEnabled = false;
    }
}

// Kayıt
builder.Services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

---

## 4. N+1 Sorgu Problemi

❌ **Yanlış Kullanım:** Döngü içinde veritabanı sorgusu yapmak.

```csharp
var categories = await _context.Categories.ToListAsync();
foreach (var category in categories)
{
    var productCount = await _context.Products
        .CountAsync(p => p.CategoryId == category.Id); // N sorgu
    category.ProductCount = productCount;
}
```

✅ **İdeal Kullanım:** Tek sorguda tüm veriyi çekin.

```csharp
var categories = await _context.Categories
    .Select(c => new CategoryDto
    {
        Id = c.Id,
        Name = c.Name,
        ProductCount = c.Products.Count
    })
    .ToListAsync(); // Tek sorgu, SQL GROUP BY

// Veya GroupBy ile
var productCounts = await _context.Products
    .GroupBy(p => p.CategoryId)
    .Select(g => new { CategoryId = g.Key, Count = g.Count() })
    .ToDictionaryAsync(x => x.CategoryId, x => x.Count);
```

---

## 5. Raw SQL'i Güvensiz Kullanmak

❌ **Yanlış Kullanım:** String interpolation ile SQL injection riski.

```csharp
var products = await _context.Products
    .FromSqlRaw($"SELECT * FROM Products WHERE Name LIKE '%{searchTerm}%'")
    .ToListAsync();
// SQL Injection: searchTerm = "'; DROP TABLE Products; --"
```

✅ **İdeal Kullanım:** Parametreli sorgular veya FromSqlInterpolated kullanın.

```csharp
// Parametreli sorgu
var products = await _context.Products
    .FromSqlInterpolated(
        $"SELECT * FROM Products WHERE Name LIKE {'%' + searchTerm + '%'}")
    .ToListAsync();

// Veya LINQ ile (daha güvenli)
var products = await _context.Products
    .Where(p => EF.Functions.Like(p.Name, $"%{searchTerm}%"))
    .ToListAsync();

// Karmaşık sorgular için stored procedure
var report = await _context.Database
    .SqlQueryRaw<SalesReport>(
        "EXEC sp_GetSalesReport @StartDate, @EndDate",
        new SqlParameter("@StartDate", startDate),
        new SqlParameter("@EndDate", endDate))
    .ToListAsync();
```
