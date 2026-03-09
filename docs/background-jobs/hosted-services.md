# Hosted Services ve BackgroundService

IHostedService ve BackgroundService, uygulamanın yaşam döngüsüne bağlı arka plan görevleri çalıştırır; yanlış kullanımlar bellek sızıntılarına ve graceful shutdown sorunlarına yol açar.

---

## 1. Scoped Servisleri Doğrudan Inject Etmek

❌ **Yanlış Kullanım:** BackgroundService içinde scoped servisi doğrudan kullanmak.

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly AppDbContext _context; // Scoped servis, captive dependency!

    public OrderProcessingService(AppDbContext context)
    {
        _context = context;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var orders = await _context.Orders.Where(o => o.Status == OrderStatus.Pending).ToListAsync(ct);
            // DbContext ömrü boyunca aynı instance kullanılır, memory leak
        }
    }
}
```

✅ **İdeal Kullanım:** IServiceScopeFactory ile her iterasyonda yeni scope oluşturun.

```csharp
public class OrderProcessingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderProcessingService> _logger;

    public OrderProcessingService(IServiceScopeFactory scopeFactory, ILogger<OrderProcessingService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

            var orders = await context.Orders
                .Where(o => o.Status == OrderStatus.Pending)
                .ToListAsync(ct);

            foreach (var order in orders)
            {
                await ProcessOrderAsync(order, scope.ServiceProvider, ct);
            }

            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}
```

---

## 2. CancellationToken'ı Kullanmamak

❌ **Yanlış Kullanım:** Uygulama kapatılırken devam eden işlemleri durdurmamak.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (true) // Kapatma sinyali kontrol edilmiyor
    {
        await ProcessPendingItemsAsync();
        Thread.Sleep(5000); // Thread.Sleep kullanmak, CancellationToken ile iptal edilemez
    }
}
```

✅ **İdeal Kullanım:** CancellationToken'ı tüm async operasyonlara iletin.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    _logger.LogInformation("Background service başlatıldı");

    while (!ct.IsCancellationRequested)
    {
        try
        {
            await ProcessPendingItemsAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
        catch (OperationCanceledException) when (ct.IsCancellationRequested)
        {
            _logger.LogInformation("Background service durduruluyor");
        }
    }
}
```

---

## 3. Hata Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Exception fırlatıldığında background service'in sessizce ölmesi.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        var items = await GetItemsAsync(ct); // Exception fırlatırsa servis durur
        foreach (var item in items)
        {
            await ProcessAsync(item, ct);    // Tek bir hata tüm servisi öldürür
        }
    }
}
```

✅ **İdeal Kullanım:** Hataları yakalayıp logla ve servisi çalışır tutun.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        try
        {
            var items = await GetItemsAsync(ct);
            foreach (var item in items)
            {
                try
                {
                    await ProcessAsync(item, ct);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Item işlenirken hata: {ItemId}", item.Id);
                }
            }
        }
        catch (Exception ex) when (!ct.IsCancellationRequested)
        {
            _logger.LogError(ex, "Background service hatası, yeniden denenecek");
            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}
```

---

## 4. ExecuteAsync'in Startup'ı Bloklaması

❌ **Yanlış Kullanım:** ExecuteAsync'de senkron işlem yaparak uygulamanın başlamasını engellemek.

```csharp
protected override Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        // Senkron çalışır, uygulama başlayamaz
        Thread.Sleep(1000);
        ProcessItems();
    }
    return Task.CompletedTask;
}
```

✅ **İdeal Kullanım:** Task.Run veya async/await ile non-blocking çalıştırın.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    // İlk await'e kadar senkron çalışır, await sonrası non-blocking
    await Task.Yield(); // Hemen yield ederek startup'ı bloklamayı önler

    while (!ct.IsCancellationRequested)
    {
        await ProcessItemsAsync(ct);
        await Task.Delay(TimeSpan.FromSeconds(1), ct);
    }
}
```

---

## 5. Timed Background Service

❌ **Yanlış Kullanım:** Manuel while-delay döngüsü ile zamanlama yapmak.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await DoWorkAsync(ct);
        await Task.Delay(TimeSpan.FromMinutes(5), ct);
        // İşlem 2 dakika sürerse aralık 7 dakika olur
    }
}
```

✅ **İdeal Kullanım:** PeriodicTimer ile düzenli aralıklarla çalıştırın.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

    while (await timer.WaitForNextTickAsync(ct))
    {
        try
        {
            using var scope = _scopeFactory.CreateScope();
            var service = scope.ServiceProvider.GetRequiredService<ICleanupService>();
            await service.RunAsync(ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Temizlik görevi başarısız");
        }
    }
}
```