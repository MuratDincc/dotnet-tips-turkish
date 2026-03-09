# Grafana ile Monitoring Dashboardları

Grafana, metrikleri görselleştirerek sistem sağlığını izlemeyi sağlar; yanlış dashboard tasarımı önemli sorunların gözden kaçmasına yol açar.

---

## 1. Tek Büyük Dashboard Oluşturmak

❌ **Yanlış Kullanım:** Tüm metrikleri tek dashboard'da toplamak.

```
// Tek dashboard: 50+ panel, yüklenmesi yavaş, kafa karıştırıcı
// CPU, Memory, HTTP istekleri, DB sorguları, Cache, Queue, Business metrikler hepsi bir arada
```

✅ **İdeal Kullanım:** Katmanlı dashboard yapısı oluşturun.

```
// Dashboard hiyerarşisi:
// 1. Overview Dashboard (üst seviye sağlık)
//    - Tüm servislerin durumu (yeşil/kırmızı)
//    - Toplam istek/hata oranı
//    - P95 latency

// 2. Service Dashboard (servis detayı)
//    - HTTP istek metrikleri (RED)
//    - Endpoint bazlı performans
//    - Dependency sağlığı

// 3. Infrastructure Dashboard (altyapı)
//    - CPU, Memory, Disk
//    - Container metrikleri
//    - Network I/O
```

```csharp
// Her katman için doğru metrikler
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()  // HTTP metrikleri
        .AddRuntimeInstrumentation()      // GC, ThreadPool
        .AddProcessInstrumentation()      // CPU, Memory
        .AddPrometheusExporter());
```

---

## 2. Variable Kullanmamak

❌ **Yanlış Kullanım:** Her servis için ayrı dashboard oluşturmak.

```
// order-service-dashboard
// payment-service-dashboard
// inventory-service-dashboard
// Aynı panel yapısı tekrarlanır, bakım zorlaşır
```

✅ **İdeal Kullanım:** Template variable ile dinamik dashboard yapın.

```json
{
  "templating": {
    "list": [
      {
        "name": "service",
        "type": "query",
        "query": "label_values(http_server_requests_total, service_name)",
        "multi": false
      },
      {
        "name": "environment",
        "type": "query",
        "query": "label_values(http_server_requests_total, deployment_environment)",
        "current": { "text": "production" }
      }
    ]
  }
}
```

```
// Grafana panel sorgusu - variable kullanımı
rate(http_server_requests_total{service_name="$service", deployment_environment="$environment"}[5m])
```

---

## 3. Alert Eşiklerini Yanlış Belirlemek

❌ **Yanlış Kullanım:** Sabit eşikler ile çok fazla false positive.

```
// Alert: CPU > 50% → her yoğun saatte uyarı gelir
// Alert: Response time > 100ms → normal varyasyonlarda bile tetiklenir
// Alert fatigue: ekip uyarıları görmezden gelmeye başlar
```

✅ **İdeal Kullanım:** Anomali bazlı ve anlamlı eşikler kullanın.

```yaml
# Grafana alert rules
- alert: HighErrorRate
  condition: rate(errors) / rate(requests) > 0.01
  for: 5m
  # %1'den fazla hata 5 dakika boyunca devam ederse

- alert: LatencyAnomaly
  condition: p95_latency > avg_over_time(p95_latency[7d]) * 2
  for: 10m
  # Son 7 günlük ortalamadan 2 kat fazla latency

- alert: SaturationWarning
  condition: container_memory_usage / container_memory_limit > 0.85
  for: 15m
  # Bellek kullanımı %85'in üzerinde
```

---

## 4. SLO Dashboard'u Oluşturmamak

❌ **Yanlış Kullanım:** Sadece teknik metrikleri izlemek.

```
// Dashboard'da sadece CPU, Memory, Request count var
// "Kullanıcı deneyimi nasıl?" sorusuna cevap yok
```

✅ **İdeal Kullanım:** SLI/SLO bazlı dashboard oluşturun.

```csharp
// SLI metrikleri tanımlama
public class SloMetrics
{
    private readonly Counter<long> _totalRequests;
    private readonly Counter<long> _successfulRequests;

    public SloMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("SLO");
        _totalRequests = meter.CreateCounter<long>("slo.requests.total");
        _successfulRequests = meter.CreateCounter<long>("slo.requests.successful");
    }

    public void Record(string sloName, bool success)
    {
        _totalRequests.Add(1, new("slo", sloName));
        if (success)
            _successfulRequests.Add(1, new("slo", sloName));
    }
}
```

```
// Grafana SLO panel sorguları
// Availability SLO: %99.9
sum(rate(slo_requests_successful_total{slo="availability"}[30d]))
/
sum(rate(slo_requests_total{slo="availability"}[30d]))

// Error Budget kalan
1 - ((1 - availability_ratio) / (1 - 0.999))
```

---

## 5. Data Source'ları Birleştirmemek

❌ **Yanlış Kullanım:** Tek kaynak ile sınırlı görünüm.

```
// Sadece Prometheus metrikleri
// Log, trace ve metrik arasında bağlantı yok
```

✅ **İdeal Kullanım:** Grafana'da çoklu data source'ları birleştirin.

```yaml
# docker-compose.yml - Grafana stack
grafana:
  image: grafana/grafana:latest
  environment:
    - GF_AUTH_ANONYMOUS_ENABLED=true
  volumes:
    - ./grafana/provisioning:/etc/grafana/provisioning

# provisioning/datasources/datasources.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true

  - name: Jaeger
    type: jaeger
    url: http://jaeger:16686

  - name: Elasticsearch
    type: elasticsearch
    url: http://elasticsearch:9200
    jsonData:
      index: "app-*"
      timeField: "@timestamp"
```

```csharp
// Exemplar desteği ile metrik → trace bağlantısı
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddPrometheusExporter(options =>
        {
            options.OpenMetricsEnabled = true; // Exemplar desteği
        }));
```
