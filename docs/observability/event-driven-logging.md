# Event-Driven Logging ve Alerting Mekanizmaları

Event-driven logging, kritik olayları tespit ederek otomatik uyarı gönderir; yanlış yapılandırma önemli olayların kaçırılmasına veya alert yorgunluğuna yol açar.

---

## 1. Alert Olmadan Çalışmak

❌ **Yanlış Kullanım:** Logları sadece kaydetmek, uyarı kurmamak.

```csharp
_logger.LogError(ex, "Ödeme işlemi başarısız: {OrderId}", orderId);
// Log yazıldı ama kimse fark etmiyor, müşteri mağdur
```

✅ **İdeal Kullanım:** Kritik olaylarda otomatik uyarı gönderin.

```csharp
public class AlertingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IAlertService _alertService;
    private readonly ILogger<AlertingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "İşlenmeyen hata: {Path}", context.Request.Path);

            if (IsCritical(ex))
                await _alertService.SendAsync(new Alert
                {
                    Severity = AlertSeverity.Critical,
                    Title = $"Kritik hata: {ex.GetType().Name}",
                    Message = ex.Message,
                    Source = context.Request.Path
                });

            throw;
        }
    }

    private static bool IsCritical(Exception ex) =>
        ex is PaymentException or DatabaseException or SecurityException;
}
```

---

## 2. Her Hatada Alert Göndermek

❌ **Yanlış Kullanım:** Tüm hatalarda bildirim göndermek.

```csharp
public class AlertService
{
    public async Task HandleErrorAsync(Exception ex)
    {
        await _slackClient.SendAsync($"HATA: {ex.Message}");
        // Günde yüzlerce mesaj, ekip uyarıları görmezden gelir
    }
}
```

✅ **İdeal Kullanım:** Throttling ve deduplication ile akıllı alerting yapın.

```csharp
public class SmartAlertService : IAlertService
{
    private readonly IDistributedCache _cache;
    private readonly ISlackClient _slack;

    public async Task SendAsync(Alert alert)
    {
        var deduplicationKey = $"alert:{alert.Title}:{alert.Source}";

        // Son 5 dakikada aynı alert gönderildi mi?
        var existing = await _cache.GetStringAsync(deduplicationKey);
        if (existing != null)
        {
            var count = int.Parse(existing) + 1;
            await _cache.SetStringAsync(deduplicationKey, count.ToString(),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
            return; // Tekrar gönderme
        }

        await _cache.SetStringAsync(deduplicationKey, "1",
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });

        await _slack.SendAsync(FormatAlert(alert));
    }
}
```

---

## 3. Alert Severity Tanımlamamak

❌ **Yanlış Kullanım:** Tüm uyarıları aynı seviyede göndermek.

```csharp
await _slack.SendAsync($"Alert: {message}");
// 404 hatası ile veritabanı çökmesi aynı kanalda, aynı formatta
```

✅ **İdeal Kullanım:** Severity bazlı routing ve escalation yapın.

```csharp
public enum AlertSeverity { Info, Warning, Critical, Fatal }

public class AlertRouter
{
    public async Task RouteAsync(Alert alert)
    {
        switch (alert.Severity)
        {
            case AlertSeverity.Info:
                await _slack.SendToChannelAsync("#monitoring-info", alert);
                break;

            case AlertSeverity.Warning:
                await _slack.SendToChannelAsync("#monitoring-warnings", alert);
                break;

            case AlertSeverity.Critical:
                await _slack.SendToChannelAsync("#incidents", alert);
                await _pagerDuty.CreateIncidentAsync(alert);
                break;

            case AlertSeverity.Fatal:
                await _slack.SendToChannelAsync("#incidents", alert);
                await _pagerDuty.CreateIncidentAsync(alert, urgency: "high");
                await _sms.SendToOnCallAsync(alert.Message);
                break;
        }
    }
}
```

---

## 4. Business Event'leri Loglamamak

❌ **Yanlış Kullanım:** Sadece teknik hataları loglamak.

```csharp
// Sadece exception loglanıyor
// Sipariş iptal oranı arttı? Ödeme başarısızlığı arttı? Bilinmiyor
```

✅ **İdeal Kullanım:** İş olaylarını da loglayın ve izleyin.

```csharp
public class OrderService
{
    public async Task<Result> CancelOrderAsync(int orderId, string reason)
    {
        var order = await _repository.GetByIdAsync(orderId);
        order.Cancel(reason);
        await _repository.SaveAsync();

        _logger.LogInformation("İş olayı: Siparişİptal {OrderId} {Reason} {CustomerId} {Amount}",
            orderId, reason, order.CustomerId, order.TotalAmount);

        _metrics.RecordBusinessEvent("order_cancelled", new()
        {
            ["reason"] = reason,
            ["customer_tier"] = order.CustomerTier
        });

        // Anormal iptal oranı kontrolü
        var cancelRate = await _metrics.GetCancelRateAsync(TimeSpan.FromHours(1));
        if (cancelRate > 0.1) // %10'dan fazla iptal
            await _alertService.SendAsync(new Alert
            {
                Severity = AlertSeverity.Warning,
                Title = "Yüksek sipariş iptal oranı",
                Message = $"Son 1 saatte iptal oranı: {cancelRate:P1}"
            });

        return Result.Success();
    }
}
```

---

## 5. Audit Log Tutmamak

❌ **Yanlış Kullanım:** Kullanıcı eylemlerini izlememek.

```csharp
public async Task DeleteUserAsync(int userId)
{
    var user = await _context.Users.FindAsync(userId);
    _context.Users.Remove(user);
    await _context.SaveChangesAsync();
    // Kim sildi? Ne zaman? Neden? Bilinmiyor
}
```

✅ **İdeal Kullanım:** Audit log ile tüm kritik eylemleri kaydedin.

```csharp
public class AuditLogService
{
    public async Task LogAsync(AuditEntry entry)
    {
        await _context.AuditLogs.AddAsync(new AuditLog
        {
            UserId = entry.UserId,
            Action = entry.Action,
            EntityType = entry.EntityType,
            EntityId = entry.EntityId,
            OldValues = JsonSerializer.Serialize(entry.OldValues),
            NewValues = JsonSerializer.Serialize(entry.NewValues),
            Timestamp = DateTimeOffset.UtcNow,
            IpAddress = entry.IpAddress
        });
        await _context.SaveChangesAsync();
    }
}

// EF Core interceptor ile otomatik audit
public class AuditSaveChangesInterceptor : SaveChangesInterceptor
{
    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct)
    {
        var context = eventData.Context!;
        var auditEntries = context.ChangeTracker.Entries()
            .Where(e => e.State is EntityState.Modified or EntityState.Deleted)
            .Select(e => new AuditEntry
            {
                EntityType = e.Entity.GetType().Name,
                Action = e.State.ToString(),
                OldValues = e.OriginalValues.Properties
                    .ToDictionary(p => p.Name, p => e.OriginalValues[p])
            }).ToList();

        // Audit kayıtlarını sakla
        return await base.SavingChangesAsync(eventData, result, ct);
    }
}
```
