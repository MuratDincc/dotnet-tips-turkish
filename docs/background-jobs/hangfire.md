# Hangfire

Hangfire, .NET uygulamalarında arka plan görevlerini kolayca planlamayı ve yönetmeyi sağlar; yanlış yapılandırma iş kayıplarına ve güvenlik açıklarına yol açar.

---

## 1. Dashboard Güvenliğini Sağlamamak

❌ **Yanlış Kullanım:** Hangfire Dashboard'ı herkese açık bırakmak.

```csharp
app.UseHangfireDashboard(); // Herkes erişebilir, jobları silebilir/tetikleyebilir
```

✅ **İdeal Kullanım:** Dashboard'a yetkilendirme filtresi ekleyin.

```csharp
public class HangfireAuthFilter : IDashboardAuthorizationFilter
{
    public bool Authorize(DashboardContext context)
    {
        var httpContext = context.GetHttpContext();
        return httpContext.User.Identity?.IsAuthenticated == true
            && httpContext.User.IsInRole("Admin");
    }
}

app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthFilter() },
    IsReadOnlyFunc = context => !context.GetHttpContext().User.IsInRole("SuperAdmin")
});
```

---

## 2. Fire-and-Forget Job'larda Hata Yönetimi

❌ **Yanlış Kullanım:** Job içinde hata yakalamamak ve retry konfigürasyonu yapmamak.

```csharp
BackgroundJob.Enqueue(() => SendEmail(userId));

public void SendEmail(int userId)
{
    var user = _context.Users.Find(userId); // User silinmişse NullRef
    _emailService.Send(user.Email, "Merhaba");
    // Hata durumunda 10 kez retry (varsayılan) - gereksiz yük
}
```

✅ **İdeal Kullanım:** Retry politikası ve hata yönetimi ekleyin.

```csharp
BackgroundJob.Enqueue<IEmailJobService>(x => x.SendWelcomeEmailAsync(userId));

public class EmailJobService : IEmailJobService
{
    [AutomaticRetry(Attempts = 3, DelaysInSeconds = new[] { 60, 300, 900 })]
    public async Task SendWelcomeEmailAsync(int userId)
    {
        var user = await _context.Users.FindAsync(userId);
        if (user == null)
        {
            _logger.LogWarning("Kullanıcı bulunamadı: {UserId}", userId);
            return; // Retry yapılmasına gerek yok
        }

        await _emailService.SendAsync(user.Email, "Hoşgeldiniz");
    }
}
```

---

## 3. Job Parametrelerinde Büyük Nesneler Geçmek

❌ **Yanlış Kullanım:** Büyük nesneleri job parametresi olarak serialize etmek.

```csharp
var orders = await _context.Orders.Include(o => o.Items).ToListAsync();
BackgroundJob.Enqueue(() => ProcessOrders(orders)); // Büyük nesne serialize edilir
```

✅ **İdeal Kullanım:** Sadece ID geçin, job içinde veriyi çekin.

```csharp
var orderIds = await _context.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .Select(o => o.Id)
    .ToListAsync();

foreach (var orderId in orderIds)
{
    BackgroundJob.Enqueue<IOrderProcessor>(x => x.ProcessAsync(orderId));
}

public class OrderProcessor : IOrderProcessor
{
    [AutomaticRetry(Attempts = 3)]
    public async Task ProcessAsync(int orderId)
    {
        var order = await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == orderId);

        if (order == null) return;
        // İşlem
    }
}
```

---

## 4. Recurring Job'larda Overlap Kontrolü

❌ **Yanlış Kullanım:** Önceki çalışma bitmeden yeni instance başlatmak.

```csharp
RecurringJob.AddOrUpdate<IReportService>(
    "daily-report",
    x => x.GenerateAsync(),
    Cron.Minutely); // Rapor 5 dakika sürüyorsa her dakika yeni bir instance başlar
```

✅ **İdeal Kullanım:** DisableConcurrentExecution ile overlap önleyin.

```csharp
RecurringJob.AddOrUpdate<IReportService>(
    "daily-report",
    x => x.GenerateAsync(),
    Cron.Daily(2, 0)); // Her gün saat 02:00

public class ReportService : IReportService
{
    [DisableConcurrentExecution(timeoutInSeconds: 300)]
    [AutomaticRetry(Attempts = 2)]
    public async Task GenerateAsync()
    {
        _logger.LogInformation("Günlük rapor oluşturma başladı");
        // Uzun süren rapor işlemi
    }
}
```

---

## 5. Job Storage'ı Yanlış Yapılandırmak

❌ **Yanlış Kullanım:** InMemory storage ile production'da çalışmak.

```csharp
builder.Services.AddHangfire(config => config.UseInMemoryStorage());
// Uygulama restart olduğunda tüm bekleyen joblar kaybolur
```

✅ **İdeal Kullanım:** Kalıcı storage ile production konfigürasyonu yapın.

```csharp
builder.Services.AddHangfire(config =>
{
    config.UseSqlServerStorage(builder.Configuration.GetConnectionString("Hangfire"), new SqlServerStorageOptions
    {
        CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
        SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
        QueuePollInterval = TimeSpan.FromSeconds(15),
        UseRecommendedIsolationLevel = true,
        DisableGlobalLocks = true,
        SchemaName = "hangfire"
    });
});

builder.Services.AddHangfireServer(options =>
{
    options.WorkerCount = Environment.ProcessorCount * 2;
    options.Queues = new[] { "critical", "default", "low" };
});
```