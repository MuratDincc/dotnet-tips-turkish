# Prometheus ile Metrik Toplama

Prometheus, zaman serisi verileri toplar ve uyarı mekanizmaları sağlar; yanlış metrik tasarımı anlamsız dashboardlara ve gözden kaçan sorunlara yol açar.

---

## 1. Metrik Tanımlamamak

❌ **Yanlış Kullanım:** Uygulama metrikleri olmadan çalışmak.

```csharp
app.MapGet("/api/orders", async (IOrderService service) =>
{
    return await service.GetAllAsync();
    // İstek sayısı, süre, hata oranı bilinmiyor
});
```

✅ **İdeal Kullanım:** Custom metrikler tanımlayıp kaydedin.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());

app.MapPrometheusScrapingEndpoint();

// Custom metrikler
public class OrderMetrics
{
    private readonly Counter<long> _ordersCreated;
    private readonly Histogram<double> _orderProcessingDuration;
    private readonly UpDownCounter<long> _activeOrders;

    public OrderMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("OrderService");
        _ordersCreated = meter.CreateCounter<long>("orders.created", "order", "Toplam oluşturulan sipariş");
        _orderProcessingDuration = meter.CreateHistogram<double>("orders.processing.duration", "ms");
        _activeOrders = meter.CreateUpDownCounter<long>("orders.active");
    }

    public void RecordOrderCreated(string status) =>
        _ordersCreated.Add(1, new KeyValuePair<string, object?>("status", status));

    public void RecordDuration(double ms) => _orderProcessingDuration.Record(ms);
}
```

---

## 2. High-Cardinality Label Kullanmak

❌ **Yanlış Kullanım:** Benzersiz değerleri label olarak eklemek.

```csharp
_requestCounter.Add(1,
    new("user_id", userId),        // Milyonlarca benzersiz değer
    new("request_id", requestId),  // Her istekte farklı
    new("timestamp", DateTime.Now.ToString()));
// Prometheus belleği tükenir, sorgu performansı çöker
```

✅ **İdeal Kullanım:** Düşük kardinaliteli, anlamlı label'lar kullanın.

```csharp
_requestCounter.Add(1,
    new("endpoint", "/api/orders"),
    new("method", "GET"),
    new("status_code", "200"),
    new("customer_tier", "premium"));
// Sınırlı sayıda benzersiz kombinasyon
```

---

## 3. RED Metodunu Uygulamamak

❌ **Yanlış Kullanım:** Anlamsız metrikler toplamak.

```csharp
var meter = meterFactory.Create("App");
meter.CreateCounter<long>("app.started");
meter.CreateCounter<long>("app.requests");
// Rate, Error, Duration bilgisi eksik
```

✅ **İdeal Kullanım:** RED (Rate, Errors, Duration) metodunu uygulayın.

```csharp
public class HttpMetrics
{
    private readonly Counter<long> _requestCount;    // Rate
    private readonly Counter<long> _errorCount;      // Errors
    private readonly Histogram<double> _duration;    // Duration

    public HttpMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("HttpServer");
        _requestCount = meter.CreateCounter<long>("http.server.requests");
        _errorCount = meter.CreateCounter<long>("http.server.errors");
        _duration = meter.CreateHistogram<double>("http.server.duration", "ms");
    }

    public void RecordRequest(string endpoint, string method, int statusCode, double durationMs)
    {
        var tags = new TagList
        {
            { "endpoint", endpoint },
            { "method", method },
            { "status_code", statusCode.ToString() }
        };

        _requestCount.Add(1, tags);
        _duration.Record(durationMs, tags);

        if (statusCode >= 500)
            _errorCount.Add(1, tags);
    }
}
```

---

## 4. Histogram Bucket'larını Ayarlamamak

❌ **Yanlış Kullanım:** Varsayılan bucket sınırları ile ölçüm yapmak.

```csharp
var histogram = meter.CreateHistogram<double>("request.duration");
// Varsayılan bucket'lar iş yüküne uygun olmayabilir
```

✅ **İdeal Kullanım:** İş yüküne uygun bucket sınırları belirleyin.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddView("http.server.duration", new ExplicitBucketHistogramConfiguration
        {
            Boundaries = new double[] { 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000 }
        })
        .AddView("db.query.duration", new ExplicitBucketHistogramConfiguration
        {
            Boundaries = new double[] { 1, 5, 10, 25, 50, 100, 500 }
        }));
```

---

## 5. Alert Kuralları Tanımlamamak

❌ **Yanlış Kullanım:** Metrikleri toplamak ama alert oluşturmamak.

```yaml
# prometheus.yml - sadece scrape, alert yok
scrape_configs:
  - job_name: 'dotnet-app'
    static_configs:
      - targets: ['app:8080']
```

✅ **İdeal Kullanım:** Anlamlı alert kuralları tanımlayın.

```yaml
# alert_rules.yml
groups:
  - name: dotnet-app-alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_errors_total[5m]) / rate(http_server_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Yüksek hata oranı (> 5%)"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_server_duration_bucket[5m])) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency 1 saniyeyi aştı"

      - alert: HighMemoryUsage
        expr: dotnet_gc_heap_size_bytes / 1024 / 1024 > 500
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Heap boyutu 500MB üzerinde"
```
