# Caching Stratejileri

Caching, uygulama performansını artırmanın ve kaynak tüketimini azaltmanın etkili bir yoludur. Ancak, yanlış kullanılan caching stratejileri performans sorunlarına, veri tutarsızlığına ve fazla bellek kullanımına yol açabilir.

---

## 1. Gereksiz Yere Büyük Verileri Cache Etmek

❌ **Yanlış Kullanım:** Büyük verileri doğrudan cache'e eklemek.

```csharp
var largeData = GetLargeData();
_memoryCache.Set("LargeData", largeData);
```

✅ **İdeal Kullanım:** Büyük verileri bölerek veya sıkıştırarak cache'e ekleyin.

```csharp
var largeData = GetLargeData();
var compressedData = CompressData(largeData);
_memoryCache.Set("CompressedLargeData", compressedData);
```

---

## 2. Cache Süresini Yanlış Ayarlamak

❌ **Yanlış Kullanım:** Sonsuz süreyle cache kullanmak.

```csharp
_memoryCache.Set("Data", data, TimeSpan.MaxValue);
```

✅ **İdeal Kullanım:** Uygun bir süre belirleyin ve gerektiğinde sliding expiration kullanın.

```csharp
_memoryCache.Set("Data", data, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
    SlidingExpiration = TimeSpan.FromMinutes(10)
});
```

---

## 3. Distributed Cache Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Tüm cache verilerini sadece bellek içinde saklamak.

```csharp
services.AddMemoryCache();
```

✅ **İdeal Kullanım:** Dağıtılmış cache kullanarak ölçeklenebilirliği artırın.

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp_";
});
```

---

## 4. Lazy Loading Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Verileri önceden yükleyip gereksiz bellek kullanımı yapmak.

```csharp
var data = GetDataFromDatabase();
_memoryCache.Set("Data", data);
```

✅ **İdeal Kullanım:** Lazy loading ile yalnızca gerektiğinde verileri yükleyin.

```csharp
_memoryCache.GetOrCreate("Data", entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30);
    return GetDataFromDatabase();
});
```

---

## 5. Cache Invalidasyonunu Yönetememek

❌ **Yanlış Kullanım:** Veri değişikliklerini cache'e yansıtmamak.

```csharp
_memoryCache.Set("User_123", user);
user.Name = "Updated Name"; // Cache güncellenmiyor
```

✅ **İdeal Kullanım:** Veri değişikliklerini cache'de güncelleyin veya invalidasyonu yönetin.

```csharp
_memoryCache.Remove("User_123");
_memoryCache.Set("User_123", updatedUser);
```

---

## 6. Hassas Verilerin Cache Edilmesi

❌ **Yanlış Kullanım:** Şifreler veya kişisel bilgileri cache'e eklemek.

```csharp
_memoryCache.Set("UserPassword", "123456");
```

✅ **İdeal Kullanım:** Hassas verileri cache'e eklemekten kaçının.

---

## 7. Cache Kullanımını Ölçümlememek

❌ **Yanlış Kullanım:** Cache hit/miss oranlarını izlememek.

✅ **İdeal Kullanım:** Cache performansını izlemek için ölçümleme yapın.

- **Örnek:** Prometheus, Grafana veya App Insights kullanarak performans verilerini toplayın.
- **Kodda Örnek:**

```csharp
if (!_memoryCache.TryGetValue("Data", out var data))
{
    data = GetDataFromDatabase();
    _memoryCache.Set("Data", data);
    Console.WriteLine("Cache miss!");
}
else
{
    Console.WriteLine("Cache hit!");
}
```