# Cache Patterns

Cache pattern'ları, verinin ne zaman ve nasıl cache'leneceğini belirler; yanlış pattern seçimi tutarsızlık ve performans sorunlarına yol açar.

---

## 1. Write-Through vs Write-Behind Karışımı

❌ **Yanlış Kullanım:** Yazma işlemlerinde tutarsız cache stratejisi.

```csharp
public async Task UpdateProductAsync(Product product)
{
    await _repository.UpdateAsync(product);
    // Bazen cache güncelleniyor, bazen siliniyor, bazen hiçbir şey yapılmıyor
    if (product.IsPopular)
        await _cache.SetAsync($"product:{product.Id}", product);
    // Tutarsız strateji
}
```

✅ **İdeal Kullanım:** Tutarlı bir write stratejisi belirleyin.

```csharp
// Write-Through: Veriyi DB ve cache'e aynı anda yaz
public async Task UpdateProductAsync(Product product)
{
    await _repository.UpdateAsync(product);
    await _cache.SetStringAsync($"product:{product.Id}",
        JsonSerializer.Serialize(product),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30) });
}

// Cache-Invalidation: Yazma işleminde cache'i sil, okumada yeniden yükle
public async Task UpdateProductAsync(Product product)
{
    await _repository.UpdateAsync(product);
    await _cache.RemoveAsync($"product:{product.Id}");
}
```

---

## 2. Cache Key Çakışması

❌ **Yanlış Kullanım:** Belirsiz ve çakışmaya açık cache key'leri.

```csharp
await _cache.SetAsync("products", data);      // Hangi filtre?
await _cache.SetAsync("user", userData);        // Hangi kullanıcı?
await _cache.SetAsync("1", orderData);          // Ne'nin 1'i?
```

✅ **İdeal Kullanım:** Yapılandırılmış ve benzersiz cache key'leri oluşturun.

```csharp
public static class CacheKeys
{
    public static string Product(int id) => $"product:{id}";
    public static string ProductList(int page, int size) => $"products:page:{page}:size:{size}";
    public static string UserProfile(int userId) => $"user:{userId}:profile";
    public static string UserOrders(int userId) => $"user:{userId}:orders";
}

await _cache.SetAsync(CacheKeys.Product(42), data);
await _cache.SetAsync(CacheKeys.ProductList(1, 20), listData);
```

---

## 3. Stale Data Toleransını Belirlememek

❌ **Yanlış Kullanım:** Her veri için aynı cache süresi kullanmak.

```csharp
// Tüm veriler 1 saat cache'leniyor
_cache.Set(key, data, TimeSpan.FromHours(1));
// Fiyat bilgisi 1 saat eski olabilir - sorunlu
// Ülke listesi 1 saatte bir yenileniyor - gereksiz
```

✅ **İdeal Kullanım:** Verinin doğasına göre farklı TTL stratejileri uygulayın.

```csharp
public static class CacheDurations
{
    public static TimeSpan Static => TimeSpan.FromHours(24);     // Ülke, şehir listeleri
    public static TimeSpan Reference => TimeSpan.FromHours(1);   // Kategori, marka
    public static TimeSpan Transactional => TimeSpan.FromMinutes(5); // Fiyat, stok
    public static TimeSpan Volatile => TimeSpan.FromSeconds(30);  // Dashboard, metrikler
}

await _cache.SetAsync(CacheKeys.Countries(), countries, CacheDurations.Static);
await _cache.SetAsync(CacheKeys.ProductPrice(id), price, CacheDurations.Transactional);
```

---

## 4. Multi-Level Cache Kullanmamak

❌ **Yanlış Kullanım:** Her istek için Redis'e gitmek.

```csharp
public async Task<Product> GetAsync(int id)
{
    var cached = await _distributedCache.GetAsync($"product:{id}"); // Her seferinde network call
    if (cached != null) return Deserialize(cached);

    var product = await _repository.GetByIdAsync(id);
    await _distributedCache.SetAsync($"product:{id}", Serialize(product));
    return product;
}
```

✅ **İdeal Kullanım:** L1 (memory) + L2 (distributed) cache katmanları kullanın.

```csharp
public class MultiLevelCache : ICacheService
{
    private readonly IMemoryCache _l1Cache;
    private readonly IDistributedCache _l2Cache;

    public async Task<T?> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan expiry)
    {
        // L1: In-Memory (hızlı)
        if (_l1Cache.TryGetValue(key, out T l1Value))
            return l1Value;

        // L2: Redis (network call)
        var l2Bytes = await _l2Cache.GetAsync(key);
        if (l2Bytes != null)
        {
            var l2Value = MessagePackSerializer.Deserialize<T>(l2Bytes);
            _l1Cache.Set(key, l2Value, TimeSpan.FromMinutes(1)); // L1'e de koy
            return l2Value;
        }

        // Cache miss: DB'den oku
        var value = await factory();
        var bytes = MessagePackSerializer.Serialize(value);

        _l1Cache.Set(key, value, TimeSpan.FromMinutes(1));
        await _l2Cache.SetAsync(key, bytes,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = expiry });

        return value;
    }
}
```

---

## 5. Cache Warming Yapmamak

❌ **Yanlış Kullanım:** Uygulama başladığında cache'in boş olması.

```csharp
// Uygulama başladığında cache boş
// İlk istekler yavaş, tüm kullanıcılar cold cache ile karşılaşır
```

✅ **İdeal Kullanım:** Uygulama başlarken sık erişilen verileri cache'e yükleyin.

```csharp
public class CacheWarmupService : IHostedService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IDistributedCache _cache;

    public async Task StartAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        // Sık erişilen verileri önceden yükle
        var categories = await context.Categories.ToListAsync(ct);
        await _cache.SetStringAsync("categories",
            JsonSerializer.Serialize(categories),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) }, ct);

        var topProducts = await context.Products.OrderByDescending(p => p.ViewCount).Take(100).ToListAsync(ct);
        foreach (var product in topProducts)
        {
            await _cache.SetStringAsync($"product:{product.Id}",
                JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30) }, ct);
        }
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```