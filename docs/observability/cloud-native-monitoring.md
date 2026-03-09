# Cloud-Native Monitoring Stratejileri

Cloud-native monitoring, bulut ortamlarında dinamik altyapıyı izlemeyi sağlar; yanlış strateji kör noktalar yaratır ve arıza tespitini geciktirir.

---

## 1. Monolitik Monitoring Yaklaşımı Kullanmak

❌ **Yanlış Kullanım:** Her servisi bağımsız izlemek.

```csharp
// Her servis kendi log dosyasına yazar
builder.Host.UseSerilog((context, config) => config
    .WriteTo.File($"logs/{serviceName}.log"));
// Servisler arası ilişki görülmez, dağıtık hata takibi imkansız
```

✅ **İdeal Kullanım:** Merkezi observability stack ile izleyin.

```csharp
// Tüm servisler için ortak telemetri yapılandırması
public static class ObservabilityExtensions
{
    public static IHostApplicationBuilder AddObservability(
        this IHostApplicationBuilder builder, string serviceName)
    {
        builder.Services.AddOpenTelemetry()
            .ConfigureResource(r => r
                .AddService(serviceName)
                .AddAttributes(new Dictionary<string, object>
                {
                    ["deployment.environment"] = builder.Environment.EnvironmentName,
                    ["host.name"] = Environment.MachineName
                }))
            .WithTracing(tracing => tracing
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddGrpcClientInstrumentation()
                .AddOtlpExporter())
            .WithMetrics(metrics => metrics
                .AddAspNetCoreInstrumentation()
                .AddRuntimeInstrumentation()
                .AddOtlpExporter());

        builder.Logging.AddOpenTelemetry(options =>
        {
            options.IncludeScopes = true;
            options.AddOtlpExporter();
        });

        return builder;
    }
}

// Kullanım - her serviste tek satır
builder.AddObservability("order-service");
```

---

## 2. Container Metriklerini İzlememek

❌ **Yanlış Kullanım:** Sadece uygulama metriklerini toplamak.

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation());
// Container CPU, memory, restart sayısı bilinmiyor
```

✅ **İdeal Kullanım:** Container ve orchestrator metriklerini de toplayın.

```yaml
# docker-compose.yml - cAdvisor ile container metrikleri
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  ports:
    - "8080:8080"
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro

# prometheus.yml
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'dotnet-apps'
    dns_sd_configs:
      - names: ['tasks.order-service', 'tasks.payment-service']
        type: 'A'
        port: 8080
```

---

## 3. Service Mesh Observability Kullanmamak

❌ **Yanlış Kullanım:** Servisler arası trafiği izlememek.

```csharp
// HTTP çağrıları yapılıyor ama aralarındaki trafik izlenmiyor
await _httpClient.PostAsync("http://payment-service/api/charge", content);
// Latency nerede? Hangi servis yavaş? Bilinmiyor
```

✅ **İdeal Kullanım:** Service mesh veya sidecar ile trafik izleyin.

```yaml
# Kubernetes + Istio ile otomatik observability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector:4317"
            - name: OTEL_SERVICE_NAME
              value: "order-service"
```

```csharp
// Kubernetes ortamında environment variable ile yapılandırma
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r
        .AddService(
            serviceName: Environment.GetEnvironmentVariable("OTEL_SERVICE_NAME") ?? "unknown",
            serviceNamespace: Environment.GetEnvironmentVariable("OTEL_SERVICE_NAMESPACE") ?? "default"))
    .WithTracing(t => t.AddOtlpExporter())
    .WithMetrics(m => m.AddOtlpExporter());
```

---

## 4. Auto-Scaling Metrikleri Tanımlamamak

❌ **Yanlış Kullanım:** Sabit instance sayısı ile çalışmak.

```yaml
# Kubernetes deployment - sabit replica
spec:
  replicas: 3
  # Yoğun saatlerde yetersiz, boş saatlerde israf
```

✅ **İdeal Kullanım:** Custom metrikler ile otomatik ölçekleme yapın.

```csharp
// Queue depth metriği ile auto-scaling
public class QueueMetrics
{
    public QueueMetrics(IMeterFactory meterFactory, IQueueService queueService)
    {
        var meter = meterFactory.Create("Queue");
        meter.CreateObservableGauge("queue.pending_messages",
            () => queueService.GetPendingCount());
    }
}
```

```yaml
# Kubernetes HPA - custom metrik ile
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_server_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

---

## 5. Chaos Testing ile Monitoring'i Doğrulamamak

❌ **Yanlış Kullanım:** Monitoring'in gerçek arızalarda çalışacağını varsaymak.

```
// Dashboard'lar var, alert'ler var ama hiç test edilmedi
// Gerçek arızada alert tetiklenmedi, dashboard yanlış veri gösterdi
```

✅ **İdeal Kullanım:** Kontrollü arıza ile monitoring'i doğrulayın.

```csharp
// Test endpoint'leri ile arıza simülasyonu
app.MapPost("/chaos/latency", (int delayMs) =>
{
    Thread.Sleep(delayMs);
    return Results.Ok("Latency injected");
}).RequireAuthorization("Admin");

app.MapPost("/chaos/error-rate", (double rate, IChaosFlagService flags) =>
{
    flags.SetErrorRate(rate);
    return Results.Ok($"Error rate: {rate:P0}");
}).RequireAuthorization("Admin");

// Middleware'da chaos flag kontrolü
if (_chaosFlags.ShouldInjectError())
    throw new Exception("Chaos: injected error");
```

```
// Game Day Checklist:
// 1. Bir servisi durdur → Alert tetiklendi mi?
// 2. DB bağlantısını kes → Circuit breaker açıldı mı?
// 3. Latency ekle → P95 dashboard'da görünüyor mu?
// 4. CPU stress test → Auto-scaling çalıştı mı?
```
