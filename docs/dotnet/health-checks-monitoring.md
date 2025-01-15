# Health Checks ve Uygulama İzleme

Health checks, bir uygulamanın veya servislerin durumunu kontrol etmek ve sorunları hızlıca tespit etmek için kullanılır. Yanlış uygulamalar health kontrollerinin güvenilirliğini azaltabilir ve problemlerin geç fark edilmesine yol açabilir.

---

## 1. Health Check Endpoint'lerinin Güvensiz Yapılandırılması

❌ **Yanlış Kullanım:** Health check endpoint'lerini herkese açık yapmak.

```csharp
app.MapHealthChecks("/health");
```

✅ **İdeal Kullanım:** Health check endpoint'lerini yetkilendirme ile koruyun.

```csharp
app.MapHealthChecks("/health").RequireAuthorization();
```

---

## 2. Basit Yanıtlar Kullanmak

❌ **Yanlış Kullanım:** Health kontrolleri için yetersiz yanıtlar.

```plaintext
Healthy
```

✅ **İdeal Kullanım:** Detaylı yanıtlar sağlayarak sorunları anlamayı kolaylaştırın.

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(entry => new
            {
                name = entry.Key,
                status = entry.Value.Status.ToString(),
                description = entry.Value.Description
            })
        });
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsync(result);
    }
});
```

---

## 3. Yetersiz Testler

❌ **Yanlış Kullanım:** Yalnızca temel kontroller yapmak.

```csharp
services.AddHealthChecks();
```

✅ **İdeal Kullanım:** Gerekli sistem bileşenlerini kontrol edin.

```csharp
services.AddHealthChecks()
    .AddSqlServer("Server=localhost;Database=MyDb;User=sa;Password=Your_password123;")
    .AddRedis("localhost:6379")
    .AddCheck("Custom Check", () =>
        HealthCheckResult.Healthy("Custom kontrol başarılı!"));
```

---

## 4. Health Check'lerin Sürekli İzlenmemesi

❌ **Yanlış Kullanım:** Health check sonuçlarını yalnızca manuel olarak kontrol etmek.

```plaintext
// Otomatik izleme yapılmıyor
```

✅ **İdeal Kullanım:** Health check sonuçlarını izleme araçları ile entegre edin.

- **Örnek:** Prometheus ve Grafana, Azure Monitor, Datadog

```csharp
services.AddHealthChecks()
    .AddCheck("Custom Check", () =>
        HealthCheckResult.Healthy("Her şey yolunda!"))
    .ForwardToPrometheus();
```

---

## 5. Timeout Sürelerini Yanlış Ayarlamak

❌ **Yanlış Kullanım:** Çok kısa veya uzun timeout süreleri kullanmak.

```csharp
services.AddHealthChecks().AddSqlServer("Connection String", timeout: TimeSpan.FromSeconds(1));
```

✅ **İdeal Kullanım:** Uygun timeout süreleri belirleyin.

```csharp
services.AddHealthChecks().AddSqlServer("Connection String", timeout: TimeSpan.FromSeconds(5));
```

---

## 6. Health Check Verilerini Loglamamak

❌ **Yanlış Kullanım:** Health check sonuçlarını loglamamak.

✅ **İdeal Kullanım:** Health kontrolü sonuçlarını loglayarak izlenebilirliği artırın.

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        foreach (var entry in report.Entries)
        {
            logger.LogInformation("Health Check: {Name}, Status: {Status}",
                entry.Key, entry.Value.Status);
        }
        await context.Response.WriteAsync("Health check completed.");
    }
});
```

---

## 7. Custom Health Check'lerin Kötü Tasarlanması

❌ **Yanlış Kullanım:** Custom health check'lerde anlamlı olmayan durumlar döndürmek.

```csharp
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        return Task.FromResult(HealthCheckResult.Unhealthy());
    }
}
```

✅ **İdeal Kullanım:** Custom health check'leri sistem durumu hakkında doğru bilgi verecek şekilde tasarlayın.

```csharp
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var isHealthy = CheckSomeCriticalResource();
        return Task.FromResult(isHealthy
            ? HealthCheckResult.Healthy("Sistem çalışıyor.")
            : HealthCheckResult.Unhealthy("Sistem kritik bir kaynakla iletişim kuramıyor."));
    }

    private bool CheckSomeCriticalResource()
    {
        // Kritik kaynağın kontrolü
        return true;
    }
}
```