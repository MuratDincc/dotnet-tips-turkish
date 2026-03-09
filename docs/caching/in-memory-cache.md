# In-Memory Cache

In-Memory cache, uygulama belleğinde veri saklayarak en hızlı erişimi sağlar; yanlış kullanımlar bellek şişmesine ve tutarsızlığa yol açar.

---

## 1. IMemoryCache Yerine Static Dictionary Kullanmak

❌ **Yanlış Kullanım:** Static dictionary ile manuel cache yapmak.

```csharp
public static class ProductCache
{
    private static readonly Dictionary<int, Product> _cache = new();

    public static Product Get(int id) => _cache.GetValueOrDefault(id);
    public static void Set(int id, Product product) => _cache[id] = product;
    // Thread-safe değil, TTL yok, bellek sınırı yok
}
```

✅ **İdeal Kullanım:** IMemoryCache ile yönetilen cache kullanın.

```csharp
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1000;
});

public class ProductService
{
    private readonly IMemoryCache _cache;

    public async Task<Product> GetAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Size = 1;
            return await _repository.GetByIdAsync(id);
        });
    }
}
```

---

## 2. Cache Size Limiti Koymamak

❌ **Yanlış Kullanım:** Sınırsız cache ile bellek tükenmesi riski.

```csharp
builder.Services.AddMemoryCache(); // Size limit yok

// Her istek yeni cache entry ekler, bellek sürekli büyür
_cache.Set($"user-session:{sessionId}", sessionData);
```

✅ **İdeal Kullanım:** Size limit ve eviction policy belirleyin.

```csharp
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 2048; // Maksimum entry sayısı
    options.CompactionPercentage = 0.25; // %75 dolduğunda %25 temizle
});

_cache.Set($"user-session:{sessionId}", sessionData, new MemoryCacheEntryOptions
{
    Size = 1,
    Priority = CacheItemPriority.Normal,
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
});
```

---

## 3. Lazy Loading ile Cache Miss Yönetimi

❌ **Yanlış Kullanım:** Her cache miss'te senkron veritabanı çağrısı.

```csharp
public Product GetProduct(int id)
{
    if (_cache.TryGetValue(id, out Product product))
        return product;

    product = _repository.GetById(id); // Senkron çağrı, thread bloklanır
    _cache.Set(id, product);
    return product;
}
```

✅ **İdeal Kullanım:** GetOrCreateAsync ile asenkron lazy loading yapın.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
        entry.RegisterPostEvictionCallback((key, value, reason, state) =>
        {
            _logger.LogInformation("Cache evicted: {Key}, Reason: {Reason}", key, reason);
        });
        return await _repository.GetByIdAsync(id);
    });
}
```

---

## 4. Multi-Server Ortamda In-Memory Cache

❌ **Yanlış Kullanım:** Birden fazla sunucuda senkronize olmayan in-memory cache.

```csharp
// Server 1: Ürün fiyatını günceller, kendi cache'ini temizler
await _cache.RemoveAsync("product:1");

// Server 2: Hala eski fiyatı gösterir, kendi cache'inden okur
var product = _cache.Get("product:1"); // Eski veri
```

✅ **İdeal Kullanım:** Multi-server ortamda IDistributedCache kullanın.

```csharp
// Development: In-Memory
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddDistributedMemoryCache();
}
else
{
    // Production: Redis
    builder.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = builder.Configuration.GetConnectionString("Redis");
        options.InstanceName = "MyApp:";
    });
}
```

---

## 5. Cache-Aside Pattern Kullanmamak

❌ **Yanlış Kullanım:** Cache ve veritabanı senkronizasyonunu her yerde tekrarlamak.

```csharp
// Her serviste aynı pattern tekrarlanıyor
var cached = await _cache.GetAsync(key);
if (cached == null)
{
    var data = await _repository.GetAsync(id);
    await _cache.SetAsync(key, data, expiry);
    return data;
}
return cached;
```

✅ **İdeal Kullanım:** Generic cache-aside helper ile tekrarı ortadan kaldırın.

```csharp
public class CacheService : ICacheService
{
    private readonly IMemoryCache _cache;

    public async Task<T> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan? expiry = null)
    {
        if (_cache.TryGetValue(key, out T cached))
            return cached;

        var value = await factory();

        var options = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = expiry ?? TimeSpan.FromMinutes(10)
        };

        _cache.Set(key, value, options);
        return value;
    }

    public void Remove(string key) => _cache.Remove(key);
}

// Kullanım
var product = await _cacheService.GetOrCreateAsync(
    $"product:{id}",
    () => _repository.GetByIdAsync(id),
    TimeSpan.FromMinutes(15));
```