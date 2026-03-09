# Recurring Jobs

Tekrarlayan görevler periyodik olarak çalışan arka plan işlemleridir; yanlış yönetim overlapping, idempotency sorunlarına ve kaynak israfına yol açar.

---

## 1. Overlapping Execution

❌ **Yanlış Kullanım:** Önceki çalışma bitmeden yeni instance başlatmak.

```csharp
// Her 5 dakikada çalış
RecurringJob.AddOrUpdate<ISyncService>("sync", x => x.SyncAsync(), "*/5 * * * *");

public class SyncService : ISyncService
{
    public async Task SyncAsync()
    {
        var records = await _api.GetAllAsync(); // 10 dakika sürebilir
        foreach (var record in records)
        {
            await _repository.UpsertAsync(record);
        }
        // 2 instance aynı anda çalışırsa duplicate veri oluşabilir
    }
}
```

✅ **İdeal Kullanım:** Distributed lock ile eşzamanlı çalışmayı önleyin.

```csharp
public class SyncService : ISyncService
{
    private readonly IDistributedLockProvider _lockProvider;

    [DisableConcurrentExecution(timeoutInSeconds: 600)]
    public async Task SyncAsync()
    {
        await using var lockHandle = await _lockProvider.TryAcquireLockAsync(
            "sync-lock", TimeSpan.FromMinutes(10));

        if (lockHandle == null)
        {
            _logger.LogWarning("Senkronizasyon zaten çalışıyor, atlanıyor");
            return;
        }

        await ExecuteSyncAsync();
    }
}
```

---

## 2. Idempotency Sağlamamak

❌ **Yanlış Kullanım:** Aynı işlemin tekrar çalışmasında yan etki oluşması.

```csharp
public async Task SendDailyNotificationsAsync()
{
    var users = await _context.Users.Where(u => u.IsActive).ToListAsync();
    foreach (var user in users)
    {
        await _emailService.SendAsync(user.Email, "Günlük bildirim");
        // Job restart olursa bazı kullanıcılara iki kez gönderilir
    }
}
```

✅ **İdeal Kullanım:** İşlem durumunu takip ederek idempotency sağlayın.

```csharp
public async Task SendDailyNotificationsAsync()
{
    var today = DateOnly.FromDateTime(DateTime.UtcNow);

    var users = await _context.Users
        .Where(u => u.IsActive)
        .Where(u => !_context.NotificationLogs
            .Any(n => n.UserId == u.Id && n.SentDate == today))
        .ToListAsync();

    foreach (var user in users)
    {
        await _emailService.SendAsync(user.Email, "Günlük bildirim");

        _context.NotificationLogs.Add(new NotificationLog
        {
            UserId = user.Id,
            SentDate = today,
            Type = "DailyNotification"
        });
        await _context.SaveChangesAsync();
    }
}
```

---

## 3. Cron İfadesini Yanlış Kullanmak

❌ **Yanlış Kullanım:** Saniye hassasiyetinde gereksiz sıklıkta çalıştırmak.

```csharp
// Her saniye çalış - gereksiz kaynak tüketimi
RecurringJob.AddOrUpdate<IHealthChecker>("health", x => x.CheckAsync(), "* * * * * *");
```

✅ **İdeal Kullanım:** İş gereksinimine uygun aralık belirleyin.

```csharp
// Her 5 dakikada health check
RecurringJob.AddOrUpdate<IHealthChecker>(
    "health-check",
    x => x.CheckAsync(),
    "*/5 * * * *");

// Her gün gece yarısı temizlik
RecurringJob.AddOrUpdate<ICleanupService>(
    "daily-cleanup",
    x => x.CleanupAsync(),
    "0 0 * * *",
    new RecurringJobOptions { TimeZone = TimeZoneInfo.FindSystemTimeZoneById("Turkey Standard Time") });

// Her Pazartesi sabah 09:00'da rapor
RecurringJob.AddOrUpdate<IReportService>(
    "weekly-report",
    x => x.GenerateWeeklyAsync(),
    "0 9 * * MON",
    new RecurringJobOptions { TimeZone = TimeZoneInfo.FindSystemTimeZoneById("Turkey Standard Time") });
```

---

## 4. Hata Durumunda Alerting Yapmamak

❌ **Yanlış Kullanım:** Başarısız recurring job'ları fark etmemek.

```csharp
public async Task ProcessDailyAsync()
{
    try
    {
        await DoWorkAsync();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Hata oluştu"); // Sadece log, kimse fark etmez
    }
}
```

✅ **İdeal Kullanım:** Global job filter ile hata bildirimi yapın.

```csharp
public class JobFailureNotificationFilter : JobFilterAttribute, IElectStateFilter
{
    public void OnStateElection(ElectStateContext context)
    {
        if (context.CandidateState is FailedState failedState)
        {
            var jobName = context.BackgroundJob.Job.Method.Name;
            var exception = failedState.Exception;

            // Alert gönder
            var notifier = context.Storage.GetConnection()
                .GetRecurringJobs()
                .Any(); // Simplified - gerçek implementasyonda IServiceProvider kullanın

            // Slack, Teams veya e-posta ile bildirim gönder
        }
    }
}

// Global filter kaydı
builder.Services.AddHangfire(config =>
{
    config.UseSqlServerStorage(connectionString);
    GlobalJobFilters.Filters.Add(new JobFailureNotificationFilter());
});
```

---

## 5. Job Monitoring ve Metrics

❌ **Yanlış Kullanım:** Job performansını izlememek.

```csharp
public async Task SyncInventoryAsync()
{
    var items = await _api.GetItemsAsync();
    foreach (var item in items)
    {
        await _repository.UpsertAsync(item);
    }
    // Ne kadar sürdü? Kaç kayıt işlendi? Bilinmiyor
}
```

✅ **İdeal Kullanım:** Job metriklerini toplayın ve izleyin.

```csharp
public class InventorySyncJob
{
    private readonly ILogger<InventorySyncJob> _logger;
    private static readonly Counter JobExecutions = Metrics.CreateCounter(
        "inventory_sync_total", "Toplam senkronizasyon sayısı",
        new CounterConfiguration { LabelNames = new[] { "status" } });

    private static readonly Histogram JobDuration = Metrics.CreateHistogram(
        "inventory_sync_duration_seconds", "Senkronizasyon süresi");

    [DisableConcurrentExecution(timeoutInSeconds: 600)]
    public async Task SyncAsync()
    {
        using var timer = JobDuration.NewTimer();
        var processedCount = 0;

        try
        {
            var items = await _api.GetItemsAsync();
            foreach (var item in items)
            {
                await _repository.UpsertAsync(item);
                processedCount++;
            }

            JobExecutions.WithLabels("success").Inc();
            _logger.LogInformation("Senkronizasyon tamamlandı: {Count} kayıt", processedCount);
        }
        catch (Exception ex)
        {
            JobExecutions.WithLabels("failure").Inc();
            _logger.LogError(ex, "Senkronizasyon hatası, {Count} kayıt işlendi", processedCount);
            throw;
        }
    }
}
```