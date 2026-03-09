# Async Performance

Asenkron programlama doğru kullanıldığında ölçeklenebilirlik sağlar; yanlış kullanımlar thread pool tükenmesine ve performans düşüşüne yol açar.

---

## 1. Async Over Sync

❌ **Yanlış Kullanım:** Senkron kodu async wrapper ile sarmak.

```csharp
public Task<int> CalculateAsync(int x)
{
    return Task.Run(() => Calculate(x)); // CPU-bound işi thread pool'a atıyor
}

public Task SaveToFileAsync(string content)
{
    File.WriteAllText("output.txt", content); // Senkron IO
    return Task.CompletedTask;
}
```

✅ **İdeal Kullanım:** Gerçekten async olan API'leri kullanın.

```csharp
public int Calculate(int x) => x * x; // CPU-bound, senkron kalsın

public async Task SaveToFileAsync(string content)
{
    await File.WriteAllTextAsync("output.txt", content); // Gerçek async IO
}
```

---

## 2. ConfigureAwait(false) Kullanmamak

❌ **Yanlış Kullanım:** Library kodunda context capture yapmak.

```csharp
// Library kodu
public async Task<Product> GetProductAsync(int id)
{
    var data = await _httpClient.GetStringAsync($"/products/{id}");
    // SynchronizationContext yakalanır, gereksiz context switch
    return JsonSerializer.Deserialize<Product>(data);
}
```

✅ **İdeal Kullanım:** Library kodunda ConfigureAwait(false) kullanın.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var data = await _httpClient.GetStringAsync($"/products/{id}").ConfigureAwait(false);
    return JsonSerializer.Deserialize<Product>(data);
}

// ASP.NET Core'da ConfigureAwait(false) gerekmez (SynchronizationContext yoktur)
// Ama shared library'lerde kullanmak iyi pratiktir
```

---

## 3. Task.Result ve .Wait() ile Deadlock

❌ **Yanlış Kullanım:** Async metodu senkron olarak beklemek.

```csharp
public Product GetProduct(int id)
{
    var product = _service.GetProductAsync(id).Result; // Deadlock riski!
    return product;
}

public void Initialize()
{
    _service.LoadDataAsync().Wait(); // Thread pool thread'ini bloklar
}
```

✅ **İdeal Kullanım:** Async'i yukarıya kadar yayın.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    return await _service.GetProductAsync(id);
}

// Eğer senkron çağrı zorunluysa (nadir durumlar)
public Product GetProduct(int id)
{
    return Task.Run(() => _service.GetProductAsync(id)).GetAwaiter().GetResult();
}
```

---

## 4. Paralel IO İşlemlerini Sıralı Yapmak

❌ **Yanlış Kullanım:** Bağımsız async çağrıları sırayla beklemek.

```csharp
public async Task<DashboardDto> GetDashboardAsync()
{
    var orders = await _orderService.GetRecentAsync();      // 200ms
    var products = await _productService.GetTopAsync();      // 150ms
    var stats = await _statsService.GetSummaryAsync();       // 300ms
    // Toplam: 650ms (sıralı)

    return new DashboardDto(orders, products, stats);
}
```

✅ **İdeal Kullanım:** Bağımsız çağrıları paralel çalıştırın.

```csharp
public async Task<DashboardDto> GetDashboardAsync()
{
    var ordersTask = _orderService.GetRecentAsync();
    var productsTask = _productService.GetTopAsync();
    var statsTask = _statsService.GetSummaryAsync();

    await Task.WhenAll(ordersTask, productsTask, statsTask);
    // Toplam: ~300ms (paralel, en yavaş kadar)

    return new DashboardDto(ordersTask.Result, productsTask.Result, statsTask.Result);
}
```

---

## 5. ValueTask Kullanmamak

❌ **Yanlış Kullanım:** Sık çağrılan ve genellikle senkron tamamlanan metodlarda Task kullanmak.

```csharp
public async Task<Product> GetCachedProductAsync(int id)
{
    if (_cache.TryGetValue(id, out Product product))
        return product; // Senkron tamamlanıyor ama Task allocation oluyor

    product = await _repository.GetByIdAsync(id);
    _cache.Set(id, product);
    return product;
}
```

✅ **İdeal Kullanım:** Çoğunlukla cache'den dönen hot path'lerde ValueTask kullanın.

```csharp
public ValueTask<Product> GetCachedProductAsync(int id)
{
    if (_cache.TryGetValue(id, out Product product))
        return ValueTask.FromResult(product); // Allocation yok

    return new ValueTask<Product>(LoadAndCacheAsync(id));
}

private async Task<Product> LoadAndCacheAsync(int id)
{
    var product = await _repository.GetByIdAsync(id);
    _cache.Set(id, product);
    return product;
}
```