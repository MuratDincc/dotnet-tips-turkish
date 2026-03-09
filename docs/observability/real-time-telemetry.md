# Real-Time Telemetry ile Performans Analizi

Gerçek zamanlı telemetri, uygulamanın anlık durumunu izleyerek proaktif müdahale imkanı sağlar; eksik veya gecikmeli telemetri sorunların büyümesine yol açar.

---

## 1. Metrikleri Batch Olarak Göndermek

❌ **Yanlış Kullanım:** Metrikleri uzun aralıklarla toplu göndermek.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddPrometheusExporter());

// prometheus.yml
// scrape_interval: 60s  → 1 dakikalık gecikme, anlık sorunlar kaçar
```

✅ **İdeal Kullanım:** Kısa aralıklarla scrape ve push yapın.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://otel-collector:4317");
            options.ExportProcessorType = ExportProcessorType.Simple;
            options.PeriodicExportingMetricReaderOptions = new()
            {
                ExportIntervalMilliseconds = 10_000 // 10 saniye
            };
        }));
```

---

## 2. Runtime Metriklerini İzlememek

❌ **Yanlış Kullanım:** Sadece HTTP metriklerini toplamak.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation());
// GC, ThreadPool, Memory bilgisi yok
```

✅ **İdeal Kullanım:** Runtime ve process metriklerini de toplayın.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddProcessInstrumentation()
        .AddMeter("Microsoft.AspNetCore.Hosting")
        .AddMeter("Microsoft.AspNetCore.Server.Kestrel")
        .AddPrometheusExporter());

// dotnet-counters ile anlık izleme (development)
// dotnet-counters monitor --process-id <pid> --counters
//   System.Runtime,
//   Microsoft.AspNetCore.Hosting,
//   Microsoft.AspNetCore.Http.Connections
```

---

## 3. Custom Metrik Oluşturmamak

❌ **Yanlış Kullanım:** Sadece framework metriklerine güvenmek.

```csharp
// Sadece varsayılan HTTP istek metrikleri
// İş metriği yok: sipariş/dk, ödeme başarı oranı, stok durumu
```

✅ **İdeal Kullanım:** İş mantığına özel metrikler tanımlayın.

```csharp
public class BusinessMetrics
{
    private readonly Counter<long> _ordersProcessed;
    private readonly Histogram<double> _orderValue;
    private readonly ObservableGauge<int> _queueDepth;

    public BusinessMetrics(IMeterFactory meterFactory, IQueueService queueService)
    {
        var meter = meterFactory.Create("Business");

        _ordersProcessed = meter.CreateCounter<long>(
            "business.orders.processed", "order", "İşlenen sipariş sayısı");

        _orderValue = meter.CreateHistogram<double>(
            "business.orders.value", "TRY", "Sipariş tutarı dağılımı");

        _queueDepth = meter.CreateObservableGauge(
            "business.queue.depth",
            () => queueService.GetPendingCount(),
            "item", "Bekleyen iş sayısı");
    }

    public void RecordOrder(decimal amount, string paymentMethod)
    {
        _ordersProcessed.Add(1, new("payment_method", paymentMethod));
        _orderValue.Record((double)amount);
    }
}
```

---

## 4. Health Check'leri Metriklerle İlişkilendirmemek

❌ **Yanlış Kullanım:** Health check ve metrikler ayrı yaşam döngüsünde.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString);

// Health check sonuçları metrik olarak izlenmiyor
```

✅ **İdeal Kullanım:** Health check sonuçlarını metrik olarak yayınlayın.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database")
    .AddRedis(redisConnectionString, name: "redis")
    .AddUrlGroup(new Uri("https://api.external.com"), name: "external-api");

// Health check metrikleri
public class HealthCheckMetricsPublisher : IHealthCheckPublisher
{
    private readonly ObservableGauge<int> _healthGauge;

    public HealthCheckMetricsPublisher(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("HealthChecks");
        _healthGauge = meter.CreateObservableGauge<int>("health.status", () => _lastResults);
    }

    private List<Measurement<int>> _lastResults = new();

    public Task PublishAsync(HealthReport report, CancellationToken ct)
    {
        _lastResults = report.Entries.Select(e => new Measurement<int>(
            e.Value.Status == HealthStatus.Healthy ? 1 : 0,
            new KeyValuePair<string, object?>("check", e.Key)
        )).ToList();

        return Task.CompletedTask;
    }
}

builder.Services.Configure<HealthCheckPublisherOptions>(options =>
{
    options.Period = TimeSpan.FromSeconds(30);
});
builder.Services.AddSingleton<IHealthCheckPublisher, HealthCheckMetricsPublisher>();
```

---

## 5. Live Dashboard Kullanmamak

❌ **Yanlış Kullanım:** .NET Aspire Dashboard'u veya dotnet-monitor'ı kullanmamak.

```csharp
// Sadece uzak monitoring araçlarına bağımlı
// Geliştirme sırasında anlık telemetri görülemiyor
```

✅ **İdeal Kullanım:** Development'ta Aspire Dashboard ile anlık izleme yapın.

```csharp
// .NET Aspire Dashboard (standalone)
// docker run -p 18888:18888 -p 4317:18889 mcr.microsoft.com/dotnet/aspire-dashboard:latest

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddOtlpExporter())
    .UseOtlpExporter(OtlpExportProtocol.Grpc,
        new Uri("http://localhost:4317"));

// http://localhost:18888 adresinden trace, metrik ve log görüntüleme
```
