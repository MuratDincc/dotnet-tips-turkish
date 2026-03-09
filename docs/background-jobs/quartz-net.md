# Quartz.NET

Quartz.NET, .NET için güçlü bir iş zamanlama kütüphanesidir; yanlış yapılandırma zamanlama hatalarına ve kaynak israfına yol açar.

---

## 1. DI Entegrasyonu Yapmamak

❌ **Yanlış Kullanım:** Job içinde bağımlılıkları manuel oluşturmak.

```csharp
public class ReportJob : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        var dbContext = new AppDbContext(); // Manuel oluşturma, DI yok
        var service = new ReportService(dbContext);
        await service.GenerateAsync();
    }
}
```

✅ **İdeal Kullanım:** Quartz DI entegrasyonu ile bağımlılıkları inject edin.

```csharp
builder.Services.AddQuartz(q =>
{
    q.UseMicrosoftDependencyInjectionJobFactory();

    q.AddJob<ReportJob>(opts => opts.WithIdentity("report-job"));
    q.AddTrigger(opts => opts
        .ForJob("report-job")
        .WithIdentity("report-trigger")
        .WithCronSchedule("0 0 2 * * ?"));  // Her gün saat 02:00
});

builder.Services.AddQuartzHostedService(q => q.WaitForJobsToComplete = true);

public class ReportJob : IJob
{
    private readonly IReportService _reportService;
    private readonly ILogger<ReportJob> _logger;

    public ReportJob(IReportService reportService, ILogger<ReportJob> logger)
    {
        _reportService = reportService;
        _logger = logger;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("Rapor oluşturma başladı");
        await _reportService.GenerateAsync(context.CancellationToken);
    }
}
```

---

## 2. Trigger Konfigürasyonunu Hardcoded Yapmak

❌ **Yanlış Kullanım:** Zamanlama ifadelerini kod içinde sabitlemek.

```csharp
q.AddTrigger(opts => opts
    .ForJob("cleanup-job")
    .WithCronSchedule("0 0 3 * * ?")); // Değiştirmek için deploy gerekir
```

✅ **İdeal Kullanım:** Konfigürasyondan zamanlamayı okuyun.

```csharp
builder.Services.AddQuartz(q =>
{
    var jobConfig = builder.Configuration.GetSection("Jobs");

    q.AddJob<CleanupJob>(opts => opts.WithIdentity("cleanup-job"));
    q.AddTrigger(opts => opts
        .ForJob("cleanup-job")
        .WithIdentity("cleanup-trigger")
        .WithCronSchedule(jobConfig["Cleanup:CronExpression"]));
});

// appsettings.json
{
    "Jobs": {
        "Cleanup": {
            "CronExpression": "0 0 3 * * ?",
            "Enabled": true
        }
    }
}
```

---

## 3. Job Persistence Kullanmamak

❌ **Yanlış Kullanım:** RAMJobStore ile job durumunu bellekte tutmak.

```csharp
builder.Services.AddQuartz(q =>
{
    q.UseInMemoryStore(); // Uygulama restart olduğunda çalışan joblar kaybolur
});
```

✅ **İdeal Kullanım:** Veritabanı persistence ile job durumunu saklanır hale getirin.

```csharp
builder.Services.AddQuartz(q =>
{
    q.UsePersistentStore(store =>
    {
        store.UseProperties = true;
        store.UseSqlServer(builder.Configuration.GetConnectionString("Quartz"));
        store.UseNewtonsoftJsonSerializer();
    });

    q.UseDefaultThreadPool(tp => tp.MaxConcurrency = 10);
});
```

---

## 4. Misfire Stratejisi Belirlememek

❌ **Yanlış Kullanım:** Misfire (kaçırılan tetikleme) durumunu yönetmemek.

```csharp
q.AddTrigger(opts => opts
    .ForJob("daily-sync")
    .WithCronSchedule("0 0 1 * * ?"));
// Uygulama gece 01:00'de kapalıysa job hiç çalışmaz
```

✅ **İdeal Kullanım:** Misfire instruction tanımlayarak kaçırılan işleri yönetin.

```csharp
q.AddTrigger(opts => opts
    .ForJob("daily-sync")
    .WithIdentity("daily-sync-trigger")
    .WithCronSchedule("0 0 1 * * ?", x =>
        x.WithMisfireHandlingInstructionFireAndProceed()) // Kaçırılan job'ı hemen çalıştır
    .StartNow());
```

---

## 5. Job Execution Context Kullanmamak

❌ **Yanlış Kullanım:** Joblar arası veri paylaşımını global state ile yapmak.

```csharp
public static class JobState
{
    public static DateTime LastRun { get; set; }
    public static int ProcessedCount { get; set; }
}

public class SyncJob : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        JobState.LastRun = DateTime.UtcNow;
        JobState.ProcessedCount++;
    }
}
```

✅ **İdeal Kullanım:** JobDataMap ile job bazlı veri saklayın.

```csharp
public class SyncJob : IJob
{
    public async Task Execute(IJobExecutionContext context)
    {
        var dataMap = context.JobDetail.JobDataMap;
        var lastRun = dataMap.GetDateTime("LastRun");

        _logger.LogInformation("Son çalışma: {LastRun}", lastRun);

        // İşlem sonrası state güncelle
        dataMap.Put("LastRun", DateTime.UtcNow);
        dataMap.Put("ProcessedCount", dataMap.GetInt("ProcessedCount") + 1);
    }
}

// Job tanımında başlangıç verileri
q.AddJob<SyncJob>(opts => opts
    .WithIdentity("sync-job")
    .UsingJobData("LastRun", DateTime.MinValue)
    .UsingJobData("ProcessedCount", 0)
    .StoreDurably());
```