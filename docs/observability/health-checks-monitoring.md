# Health Checks ile Servis Durumu İzleme

Health check'ler, servisin ve bağımlılıklarının sağlığını izler; eksik veya yanlış health check'ler arızalı servislere trafik yönlenmesine yol açar.

---

## 1. Health Check Eklememek

❌ **Yanlış Kullanım:** Servis sağlığını izlememek.

```csharp
var app = builder.Build();
app.MapGet("/health", () => "OK");
// Veritabanı çökmüş olsa bile "OK" döner
```

✅ **İdeal Kullanım:** Bağımlılık bazlı health check'ler ekleyin.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database", tags: new[] { "db", "critical" })
    .AddRedis(redisConnectionString, name: "redis", tags: new[] { "cache" })
    .AddUrlGroup(new Uri("https://api.payment.com/health"), name: "payment-api",
        tags: new[] { "external" })
    .AddCheck<DiskSpaceHealthCheck>("disk-space", tags: new[] { "infra" });

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("critical"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Sadece uygulama ayakta mı kontrolü
});
```

---

## 2. Liveness ve Readiness Ayrımı Yapmamak

❌ **Yanlış Kullanım:** Tek health endpoint ile tüm kontrolleri yapmak.

```csharp
app.MapHealthChecks("/health");
// Kubernetes: DB yavaşsa pod restart edilir (yanlış!)
// DB sorunu restart ile çözülmez, trafik kesilmeli
```

✅ **İdeal Kullanım:** Liveness ve readiness probe'larını ayırın.

```csharp
// Liveness: Uygulama çalışıyor mu? (restart kararı)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("self"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Readiness: Trafik alabilir mi? (trafik yönlendirme kararı)
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("dependency"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Startup: Başlangıç tamamlandı mı?
app.MapHealthChecks("/health/startup", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("startup")
});
```

```yaml
# Kubernetes probes
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## 3. Custom Health Check Yazmamak

❌ **Yanlış Kullanım:** Sadece built-in check'leri kullanmak.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString);
// Disk alanı, kuyruk derinliği, lisans durumu gibi kontroller yok
```

✅ **İdeal Kullanım:** İş mantığına özel health check'ler yazın.

```csharp
public class DiskSpaceHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct = default)
    {
        var drive = new DriveInfo("/");
        var freePercentage = (double)drive.AvailableFreeSpace / drive.TotalSize * 100;

        return Task.FromResult(freePercentage switch
        {
            < 5 => HealthCheckResult.Unhealthy($"Disk alanı kritik: {freePercentage:F1}%"),
            < 15 => HealthCheckResult.Degraded($"Disk alanı düşük: {freePercentage:F1}%"),
            _ => HealthCheckResult.Healthy($"Disk alanı yeterli: {freePercentage:F1}%")
        });
    }
}

public class QueueDepthHealthCheck : IHealthCheck
{
    private readonly IQueueService _queue;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, CancellationToken ct = default)
    {
        var depth = await _queue.GetPendingCountAsync(ct);

        return depth switch
        {
            > 10000 => HealthCheckResult.Unhealthy($"Kuyruk çok dolu: {depth}"),
            > 1000 => HealthCheckResult.Degraded($"Kuyruk birikiyor: {depth}"),
            _ => HealthCheckResult.Healthy($"Kuyruk normal: {depth}")
        };
    }
}
```

---

## 4. Health Check Timeout Ayarlamamak

❌ **Yanlış Kullanım:** Health check'lerin uzun süre beklemesi.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString);
// DB yanıt vermezse health check 30sn bekler
// Bu sürede Kubernetes podu unhealthy sayar ve restart eder
```

✅ **İdeal Kullanım:** Kısa timeout ile hızlı sağlık kontrolü yapın.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database",
        timeout: TimeSpan.FromSeconds(3))
    .AddRedis(redisConnectionString, name: "redis",
        timeout: TimeSpan.FromSeconds(2))
    .AddUrlGroup(new Uri("https://api.external.com/health"),
        name: "external-api",
        timeout: TimeSpan.FromSeconds(5));

// Global timeout
builder.Services.Configure<HealthCheckServiceOptions>(options =>
{
    options.Registrations.ToList().ForEach(r =>
    {
        if (r.Timeout == default)
            r.Timeout = TimeSpan.FromSeconds(5);
    });
});
```

---

## 5. Health Check UI Kullanmamak

❌ **Yanlış Kullanım:** Health check sonuçlarını sadece JSON olarak döndürmek.

```csharp
app.MapHealthChecks("/health");
// JSON çıktısı okunması zor, geçmiş verisi yok
```

✅ **İdeal Kullanım:** Health Check UI ile görsel izleme yapın.

```csharp
builder.Services
    .AddHealthChecksUI(options =>
    {
        options.SetEvaluationTimeInSeconds(30);
        options.MaximumHistoryEntriesPerEndpoint(50);
        options.AddHealthCheckEndpoint("Order Service", "http://localhost:5000/health/ready");
        options.AddHealthCheckEndpoint("Payment Service", "http://localhost:5001/health/ready");

        options.AddWebhookNotification("Slack", "https://hooks.slack.com/services/xxx",
            payload: "{ \"text\": \"[[LIVENESS]] is [[FAILURE]]: [[DESCRIPTIONS]]\" }",
            restorePayload: "{ \"text\": \"[[LIVENESS]] is recovered\" }");
    })
    .AddInMemoryStorage();

app.MapHealthChecksUI(options =>
{
    options.UIPath = "/health-ui";
});
```
