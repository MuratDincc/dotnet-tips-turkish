# Jaeger ile Dağıtık Tracing

Jaeger, dağıtık sistemlerde istek izleme ve performans analizi sağlar; yanlış konfigürasyon trace kaybına ve analiz zorluğuna yol açar.

---

## 1. Jaeger'ı Doğrudan Exporter ile Kullanmak

❌ **Yanlış Kullanım:** Her servisten doğrudan Jaeger'a trace göndermek.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddJaegerExporter(o =>
        {
            o.AgentHost = "jaeger";
            o.AgentPort = 6831;
        }));
// Her servis ayrı bağlantı kurar, Jaeger agent yükü artar
```

✅ **İdeal Kullanım:** OTLP Collector üzerinden Jaeger'a gönderin.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(ResourceBuilder.CreateDefault()
            .AddService("order-service", serviceVersion: "1.0.0")
            .AddAttributes(new Dictionary<string, object>
            {
                ["deployment.environment"] = builder.Environment.EnvironmentName
            }))
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://otel-collector:4317")));
```

```yaml
# otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [jaeger]
```

---

## 2. Service Bağımlılık Grafiğini Oluşturamamak

❌ **Yanlış Kullanım:** Servis adlarını tutarsız tanımlamak.

```csharp
// Servis 1
.AddService("OrderSvc")
// Servis 2
.AddService("order-service")
// Servis 3
.AddService("Order_Service")
// Jaeger'da 3 farklı servis görünür, dependency graph bozulur
```

✅ **İdeal Kullanım:** Tutarlı isimlendirme convention'ı kullanın.

```csharp
public static class ServiceInfo
{
    public const string Name = "order-service";
    public const string Version = "1.0.0";
}

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetResourceBuilder(ResourceBuilder.CreateDefault()
            .AddService(ServiceInfo.Name, serviceVersion: ServiceInfo.Version)));
```

---

## 3. Span Adlarını Yanlış Vermek

❌ **Yanlış Kullanım:** Span adlarında dinamik değer kullanmak.

```csharp
using var activity = Source.StartActivity($"GET /api/orders/{orderId}");
// Her orderId için farklı span adı → Jaeger'da gruplama yapılamaz
```

✅ **İdeal Kullanım:** Span adını sabit tutup detayları attribute olarak ekleyin.

```csharp
using var activity = Source.StartActivity("GET /api/orders/{id}");
activity?.SetTag("order.id", orderId);
activity?.SetTag("http.method", "GET");
activity?.SetTag("http.route", "/api/orders/{id}");
// Jaeger'da aynı endpoint'in tüm çağrıları gruplanır
```

---

## 4. Trace Verilerini Saklama Süresini Ayarlamamak

❌ **Yanlış Kullanım:** Varsayılan retention ile disk alanı tükenmesi.

```yaml
# docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:latest
  # Varsayılan in-memory storage, restart'ta veri kaybolur
```

✅ **İdeal Kullanım:** Production-ready storage ve retention ayarlayın.

```yaml
# docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:latest
  environment:
    - SPAN_STORAGE_TYPE=elasticsearch
    - ES_SERVER_URLS=http://elasticsearch:9200
    - ES_MAX_SPAN_AGE=168h  # 7 gün
    - ES_NUM_SHARDS=3
    - ES_NUM_REPLICAS=1

elasticsearch:
  image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
  environment:
    - discovery.type=single-node
    - xpack.security.enabled=false
  volumes:
    - es-data:/usr/share/elasticsearch/data
```

---

## 5. Trace'leri Loglarla İlişkilendirmemek

❌ **Yanlış Kullanım:** Log ve trace ayrı sistemlerde, ilişki yok.

```csharp
_logger.LogError("Sipariş oluşturulamadı: {OrderId}", orderId);
// Log'dan trace'e, trace'den log'a geçiş yapılamaz
```

✅ **İdeal Kullanım:** Trace ID'yi loglara otomatik ekleyin.

```csharp
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeScopes = true;
    options.IncludeFormattedMessage = true;
    options.AddOtlpExporter();
});

// Serilog ile
builder.Host.UseSerilog((context, config) => config
    .Enrich.WithProperty("ServiceName", "order-service")
    .Enrich.With<ActivityEnricher>()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level}] {Message} TraceId={TraceId} SpanId={SpanId}{NewLine}"));

public class ActivityEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory factory)
    {
        var activity = Activity.Current;
        if (activity is null) return;

        logEvent.AddPropertyIfAbsent(factory.CreateProperty("TraceId", activity.TraceId));
        logEvent.AddPropertyIfAbsent(factory.CreateProperty("SpanId", activity.SpanId));
    }
}
```
