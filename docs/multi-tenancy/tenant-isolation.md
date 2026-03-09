# Tenant Isolation ve Data Separation

Multi-tenancy, tek uygulama üzerinde birden fazla müşteriyi izole şekilde çalıştırır; yanlış izolasyon veri sızıntısına ve güvenlik ihlallerine yol açar.

---

## 1. Global Query Filter Kullanmamak

❌ **Yanlış Kullanım:** Her sorguda manuel tenant filtresi eklemek.

```csharp
public async Task<List<Product>> GetAllAsync()
{
    var tenantId = _tenantService.GetCurrentTenantId();
    return await _context.Products
        .Where(p => p.TenantId == tenantId) // Her sorguda unutulabilir
        .ToListAsync();
}

// Bir geliştirici filtre eklemeyi unutursa başka tenant'ın verileri sızar
public async Task<List<Order>> GetOrdersAsync()
{
    return await _context.Orders.ToListAsync(); // TenantId filtresi yok!
}
```

✅ **İdeal Kullanım:** EF Core global query filter ile otomatik filtreleme yapın.

```csharp
public interface ITenantEntity
{
    string TenantId { get; set; }
}

public class MultiTenantDbContext : DbContext
{
    private readonly ITenantService _tenantService;

    public MultiTenantDbContext(DbContextOptions options, ITenantService tenantService)
        : base(options) => _tenantService = tenantService;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Tüm tenant entity'lerine otomatik filtre
        foreach (var entityType in modelBuilder.Model.GetEntityTypes())
        {
            if (typeof(ITenantEntity).IsAssignableFrom(entityType.ClrType))
            {
                var method = typeof(MultiTenantDbContext)
                    .GetMethod(nameof(SetTenantFilter), BindingFlags.NonPublic | BindingFlags.Static)!
                    .MakeGenericMethod(entityType.ClrType);

                method.Invoke(null, new object[] { modelBuilder, _tenantService });
            }
        }
    }

    private static void SetTenantFilter<T>(ModelBuilder modelBuilder, ITenantService tenantService)
        where T : class, ITenantEntity
    {
        modelBuilder.Entity<T>().HasQueryFilter(e => e.TenantId == tenantService.GetCurrentTenantId());
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<ITenantEntity>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantService.GetCurrentTenantId();
        }
        return base.SaveChangesAsync(ct);
    }
}
```

---

## 2. Tenant Resolution Yapmamak

❌ **Yanlış Kullanım:** Tenant bilgisini hardcode etmek veya güvensiz almak.

```csharp
app.MapGet("/api/products", async (HttpContext context, AppDbContext db) =>
{
    var tenantId = context.Request.Query["tenantId"].ToString(); // URL'den alınabilir, manipüle edilebilir
    return await db.Products.Where(p => p.TenantId == tenantId).ToListAsync();
});
```

✅ **İdeal Kullanım:** Middleware ile güvenli tenant resolution yapın.

```csharp
public interface ITenantService
{
    string GetCurrentTenantId();
    TenantInfo GetCurrentTenant();
}

public class TenantMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context, ITenantResolver resolver)
    {
        var tenant = await resolver.ResolveAsync(context);

        if (tenant is null)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { error = "Geçersiz tenant" });
            return;
        }

        context.Items["Tenant"] = tenant;
        await _next(context);
    }
}

public class TenantResolver : ITenantResolver
{
    private readonly ITenantRepository _repository;

    public async Task<TenantInfo?> ResolveAsync(HttpContext context)
    {
        // Strateji 1: Subdomain
        var host = context.Request.Host.Host;
        var subdomain = host.Split('.').FirstOrDefault();

        // Strateji 2: Header
        // var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();

        // Strateji 3: Claim (JWT)
        // var tenantId = context.User.FindFirst("tenant_id")?.Value;

        if (string.IsNullOrEmpty(subdomain)) return null;

        return await _repository.GetBySubdomainAsync(subdomain);
    }
}

app.UseMiddleware<TenantMiddleware>();
```

---

## 3. Tek Veritabanında İzolasyon Sağlamamak

❌ **Yanlış Kullanım:** Tenant verilerini karıştırmak.

```csharp
// Tüm tenant'lar aynı tabloda, index yok
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string TenantId { get; set; } // Index yok, performans sorunu
}
```

✅ **İdeal Kullanım:** Veritabanı stratejisini tenant ihtiyacına göre seçin.

```csharp
// Strateji 1: Paylaşımlı veritabanı, ayrı şema (orta izolasyon)
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var schema = _tenantService.GetCurrentTenant().Schema;
    modelBuilder.HasDefaultSchema(schema);
}

// Strateji 2: Ayrı veritabanı (yüksek izolasyon)
public class TenantDbContextFactory
{
    public AppDbContext CreateContext(string tenantId)
    {
        var tenant = _tenantRepository.GetById(tenantId);
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(tenant.ConnectionString)
            .Options;
        return new AppDbContext(options, _tenantService);
    }
}

// Strateji 3: Paylaşımlı tablo, Row-Level Security (düşük maliyet)
// SQL Server RLS
// CREATE SECURITY POLICY TenantPolicy
//   ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON dbo.Products
//   WITH (STATE = ON);

// EF Core kayıt
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    var tenant = sp.GetRequiredService<ITenantService>().GetCurrentTenant();
    options.UseSqlServer(tenant.ConnectionString);
});
```

---

## 4. Tenant Bazlı Konfigürasyon Yapmamak

❌ **Yanlış Kullanım:** Tüm tenant'lara aynı ayarları uygulamak.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        context => RateLimitPartition.GetFixedWindowLimiter("global",
            _ => new FixedWindowRateLimiterOptions { PermitLimit = 100 }));
});
// Küçük tenant ile enterprise tenant aynı limitlere tabi
```

✅ **İdeal Kullanım:** Tenant bazlı konfigürasyon ve limit yönetimi yapın.

```csharp
public record TenantInfo
{
    public string Id { get; init; }
    public string Name { get; init; }
    public TenantPlan Plan { get; init; }
    public int RateLimit { get; init; }
    public int MaxUsers { get; init; }
    public int MaxStorageMb { get; init; }
}

builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        context =>
        {
            var tenant = context.Items["Tenant"] as TenantInfo;
            var limit = tenant?.RateLimit ?? 10;

            return RateLimitPartition.GetFixedWindowLimiter(
                tenant?.Id ?? "anonymous",
                _ => new FixedWindowRateLimiterOptions
                {
                    PermitLimit = limit,
                    Window = TimeSpan.FromMinutes(1)
                });
        });
});
```

---

## 5. Cache'te Tenant İzolasyonu Yapmamak

❌ **Yanlış Kullanım:** Cache key'lerde tenant ayrımı yapmamak.

```csharp
public async Task<List<Product>> GetProductsAsync()
{
    var cacheKey = "products_all";
    var cached = await _cache.GetAsync<List<Product>>(cacheKey);
    if (cached != null) return cached;

    var products = await _context.Products.ToListAsync();
    await _cache.SetAsync(cacheKey, products);
    return products;
    // Tüm tenant'lar aynı cache'i paylaşır, veri sızıntısı!
}
```

✅ **İdeal Kullanım:** Cache key'lere tenant prefix ekleyin.

```csharp
public class TenantCacheService : ICacheService
{
    private readonly IDistributedCache _cache;
    private readonly ITenantService _tenantService;

    public async Task<T?> GetAsync<T>(string key)
    {
        var tenantKey = BuildKey(key);
        var data = await _cache.GetStringAsync(tenantKey);
        return data is null ? default : JsonSerializer.Deserialize<T>(data);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var tenantKey = BuildKey(key);
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry ?? TimeSpan.FromMinutes(10)
        };
        await _cache.SetStringAsync(tenantKey, JsonSerializer.Serialize(value), options);
    }

    private string BuildKey(string key)
    {
        var tenantId = _tenantService.GetCurrentTenantId();
        return $"tenant:{tenantId}:{key}";
    }
}
```
