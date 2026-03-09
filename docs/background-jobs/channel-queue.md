# Channel-Based Queue

System.Threading.Channels, producer-consumer senaryoları için yüksek performanslı asenkron kuyruk sağlar; yanlış kullanımlar bellek sorunlarına ve veri kaybına yol açar.

---

## 1. Unbounded Channel ile Bellek Sorunu

❌ **Yanlış Kullanım:** Sınırsız kapasite ile bellek tükenmesi riski oluşturmak.

```csharp
var channel = Channel.CreateUnbounded<WorkItem>();

// Producer hızlı, consumer yavaşsa kuyruk sınırsız büyür
while (true)
{
    var item = await GetNextItemAsync();
    await channel.Writer.WriteAsync(item); // Bellek tükenene kadar yazılır
}
```

✅ **İdeal Kullanım:** Bounded Channel ile kapasite limiti ve backpressure uygulayın.

```csharp
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait, // Kuyruk dolunca producer bekler
    SingleReader = false,
    SingleWriter = false
});

// DI kaydı
builder.Services.AddSingleton(Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait
}));
```

---

## 2. Background Worker ile Channel Entegrasyonu

❌ **Yanlış Kullanım:** Controller'da uzun süren işlemi senkron yapmak.

```csharp
[HttpPost("api/reports")]
public async Task<IActionResult> GenerateReport(ReportRequest request)
{
    var report = await _reportService.GenerateAsync(request); // 30 saniye sürer
    return Ok(report); // İstemci bekler, timeout olabilir
}
```

✅ **İdeal Kullanım:** Channel ile isteği kuyruğa atıp hemen yanıt dönün.

```csharp
[HttpPost("api/reports")]
public async Task<IActionResult> GenerateReport(ReportRequest request)
{
    var jobId = Guid.NewGuid();
    await _channel.Writer.WriteAsync(new ReportJob(jobId, request));
    return Accepted(new { JobId = jobId, StatusUrl = $"/api/reports/status/{jobId}" });
}

public class ReportWorker : BackgroundService
{
    private readonly Channel<ReportJob> _channel;
    private readonly IServiceScopeFactory _scopeFactory;

    public ReportWorker(Channel<ReportJob> channel, IServiceScopeFactory scopeFactory)
    {
        _channel = channel;
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in _channel.Reader.ReadAllAsync(ct))
        {
            using var scope = _scopeFactory.CreateScope();
            var service = scope.ServiceProvider.GetRequiredService<IReportService>();
            await service.GenerateAsync(job.Request, ct);
        }
    }
}
```

---

## 3. Tek Consumer ile Düşük Throughput

❌ **Yanlış Kullanım:** Tek consumer ile kuyruğu yavaş işlemek.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    await foreach (var item in _channel.Reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item); // Tek seferde bir item işlenir
    }
}
```

✅ **İdeal Kullanım:** Birden fazla consumer ile paralel işleme yapın.

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    var workerCount = Environment.ProcessorCount;
    var tasks = Enumerable.Range(0, workerCount)
        .Select(_ => ProcessChannelAsync(ct));

    await Task.WhenAll(tasks);
}

private async Task ProcessChannelAsync(CancellationToken ct)
{
    await foreach (var item in _channel.Reader.ReadAllAsync(ct))
    {
        try
        {
            using var scope = _scopeFactory.CreateScope();
            var processor = scope.ServiceProvider.GetRequiredService<IItemProcessor>();
            await processor.ProcessAsync(item, ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Item işlenirken hata: {ItemId}", item.Id);
        }
    }
}
```

---

## 4. Channel Completion'ı Yönetmemek

❌ **Yanlış Kullanım:** Channel'ı kapatmadan uygulamayı sonlandırmak.

```csharp
public class ImportService
{
    public async Task ImportAsync(IEnumerable<Record> records)
    {
        foreach (var record in records)
        {
            await _channel.Writer.WriteAsync(record);
        }
        // Writer tamamlandı ama consumer hala bekliyor
    }
}
```

✅ **İdeal Kullanım:** Writer tamamlandığında Complete ile bildirin.

```csharp
public class ImportService
{
    public async Task ImportAsync(IEnumerable<Record> records, CancellationToken ct)
    {
        try
        {
            foreach (var record in records)
            {
                await _channel.Writer.WriteAsync(record, ct);
            }
        }
        finally
        {
            _channel.Writer.Complete();
        }
    }
}

// Consumer
protected override async Task ExecuteAsync(CancellationToken ct)
{
    await foreach (var item in _channel.Reader.ReadAllAsync(ct))
    {
        await ProcessAsync(item, ct);
    }
    // Writer Complete() çağrıldığında döngü otomatik biter
    _logger.LogInformation("Tüm kayıtlar işlendi");
}
```

---

## 5. FullMode Stratejisini Yanlış Seçmek

❌ **Yanlış Kullanım:** Önemli mesajları DropOldest ile kaybetmek.

```csharp
var channel = Channel.CreateBounded<PaymentEvent>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.DropOldest // Ödeme event'leri kaybolabilir!
});
```

✅ **İdeal Kullanım:** İş gereksinimine göre uygun FullMode seçin.

```csharp
// Kritik mesajlar için: Wait (backpressure uygula)
var paymentChannel = Channel.CreateBounded<PaymentEvent>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.Wait // Producer bekler, mesaj kaybolmaz
});

// Telemetri/metrik gibi kayıp tolere edilebilir veriler için: DropOldest
var metricsChannel = Channel.CreateBounded<MetricEvent>(new BoundedChannelOptions(500)
{
    FullMode = BoundedChannelFullMode.DropOldest // Eski metrikler atılabilir
});
```