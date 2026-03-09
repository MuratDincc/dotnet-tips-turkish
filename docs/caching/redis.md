# Redis Cache

Redis, yüksek performanslı dağıtık cache çözümü sunar; yanlış kullanımlar cache tutarsızlığına ve bellek sorunlarına yol açar.

---

## 1. Cache Expiry Belirlememek

❌ **Yanlış Kullanım:** Cache'e veri yazarken TTL belirlememek.

```csharp
await _cache.SetStringAsync("product:1", JsonSerializer.Serialize(product));
// Süresiz cache, eski veri sonsuza kadar kalır
```

✅ **İdeal Kullanım:** Verinin doğasına uygun TTL belirleyin.

```csharp
await _cache.SetStringAsync("product:1", JsonSerializer.Serialize(product),
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
        SlidingExpiration = TimeSpan.FromMinutes(10)
    });
```

---

## 2. Cache Stampede (Thundering Herd)

❌ **Yanlış Kullanım:** Cache expire olduğunda tüm isteklerin veritabanına gitmesi.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cached = await _cache.GetStringAsync($"product:{id}");
    if (cached != null) return JsonSerializer.Deserialize<Product>(cached);

    // Cache boşaldığında 1000 istek aynı anda DB'ye gider
    var product = await _repository.GetByIdAsync(id);
    await _cache.SetStringAsync($"product:{id}", JsonSerializer.Serialize(product));
    return product;
}
```

✅ **İdeal Kullanım:** Lock mekanizması ile tek istek DB'ye gitsin.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var cacheKey = $"product:{id}";
    var cached = await _cache.GetStringAsync(cacheKey);
    if (cached != null) return JsonSerializer.Deserialize<Product>(cached);

    var lockKey = $"lock:{cacheKey}";
    await using var lockHandle = await _lockProvider.TryAcquireLockAsync(lockKey, TimeSpan.FromSeconds(10));

    if (lockHandle != null)
    {
        // Double-check
        cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null) return JsonSerializer.Deserialize<Product>(cached);

        var product = await _repository.GetByIdAsync(id);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30) });
        return product;
    }

    // Lock alınamazsa kısa bekleyip cache'den oku
    await Task.Delay(100);
    cached = await _cache.GetStringAsync(cacheKey);
    return cached != null ? JsonSerializer.Deserialize<Product>(cached) : await _repository.GetByIdAsync(id);
}
```

---

## 3. Büyük Nesneleri Cache'lemek

❌ **Yanlış Kullanım:** Tüm ilişkili verileri tek cache entry'sine yazmak.

```csharp
var orders = await _context.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .Include(o => o.Payments)
    .ToListAsync();

await _cache.SetStringAsync("all-orders", JsonSerializer.Serialize(orders));
// Megabyte'larca veri, serialization yavaş, bellek israfı
```

✅ **İdeal Kullanım:** Küçük ve odaklı cache entry'leri oluşturun.

```csharp
// Sadece gerekli veriyi cache'leyin
var orderSummaries = await _context.Orders
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        TotalPrice = o.TotalPrice,
        Status = o.Status
    })
    .ToListAsync();

await _cache.SetStringAsync("order-summaries",
    JsonSerializer.Serialize(orderSummaries),
    new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
```

---

## 4. Cache Invalidation Yapmamak

❌ **Yanlış Kullanım:** Veri güncellendiğinde cache'i temizlememek.

```csharp
public async Task UpdateProductAsync(Product product)
{
    _context.Products.Update(product);
    await _context.SaveChangesAsync();
    // Cache'deki eski veri hala duruyor, kullanıcılar eski veriyi görür
}
```

✅ **İdeal Kullanım:** Veri değiştiğinde ilgili cache'i invalidate edin.

```csharp
public async Task UpdateProductAsync(Product product)
{
    _context.Products.Update(product);
    await _context.SaveChangesAsync();

    await _cache.RemoveAsync($"product:{product.Id}");
    await _cache.RemoveAsync("product-list"); // Liste cache'ini de temizle
}
```

---

## 5. Serialization Performansını İhmal Etmek

❌ **Yanlış Kullanım:** Her cache işleminde yavaş JSON serialization kullanmak.

```csharp
var json = JsonSerializer.Serialize(product);
await _cache.SetStringAsync(key, json);
var cached = JsonSerializer.Deserialize<Product>(await _cache.GetStringAsync(key));
```

✅ **İdeal Kullanım:** MessagePack veya MemoryPack ile hızlı serialization yapın.

```csharp
public class RedisCacheService : ICacheService
{
    public async Task SetAsync<T>(string key, T value, TimeSpan expiry)
    {
        var bytes = MessagePackSerializer.Serialize(value);
        await _cache.SetAsync(key, bytes,
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = expiry });
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var bytes = await _cache.GetAsync(key);
        if (bytes == null) return default;
        return MessagePackSerializer.Deserialize<T>(bytes);
    }
}
```