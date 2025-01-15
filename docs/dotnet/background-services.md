# Background Services: Kötü ve İdeal Kullanım

Background services, uzun süreli işlemleri veya arka planda çalışması gereken görevleri yönetmek için kullanılır. Yanlış tasarlanmış arka plan servisleri performans sorunlarına, kaynak tüketimine ve beklenmedik hatalara yol açabilir.

---

## 1. Uzun Süreli İşlemleri UI Thread'de Çalıştırmak

❌ **Yanlış Kullanım:** Uzun süreli işlemleri UI thread'inde çalıştırmak.

```csharp
public void DoWork()
{
    Thread.Sleep(5000); // UI thread'i bloke eder
}
```

✅ **İdeal Kullanım:** Uzun süreli işlemleri arka planda çalıştırın.

```csharp
public class BackgroundTaskService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(5000, stoppingToken); // Arka planda çalışır
        }
    }
}
```

---

## 2. CancellationToken Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** İptal token'ı olmadan arka plan görevleri oluşturmak.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (true)
    {
        await Task.Delay(1000); // İptal edilemez
    }
}
```

✅ **İdeal Kullanım:** `CancellationToken` kullanarak görevlerin düzgün bir şekilde iptal edilmesini sağlayın.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await Task.Delay(1000, stoppingToken);
    }
}
```

---

## 3. Hataları Kontrolsüz Bırakmak

❌ **Yanlış Kullanım:** Arka plan görevlerindeki hataları göz ardı etmek.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        throw new Exception("Bir hata oluştu."); // Yönetilmez
    }
}
```

✅ **İdeal Kullanım:** Hataları yakalayarak loglayın ve yönetilebilir hale getirin.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            // İş mantığı
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Hata: {ex.Message}");
        }
    }
}
```

---

## 4. Fazla Sayıda Background Service Oluşturmak

❌ **Yanlış Kullanım:** Her görev için ayrı bir background service tanımlamak.

```plaintext
services.AddHostedService<Service1>();
services.AddHostedService<Service2>();
services.AddHostedService<Service3>();
```

✅ **İdeal Kullanım:** Görevleri birleştirerek yönetilebilir bir yapı oluşturun.

```csharp
public class CombinedBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var task1 = Task1(stoppingToken);
        var task2 = Task2(stoppingToken);
        await Task.WhenAll(task1, task2);
    }

    private async Task Task1(CancellationToken stoppingToken) { /* ... */ }
    private async Task Task2(CancellationToken stoppingToken) { /* ... */ }
}
```

---

## 5. Kaynakları Serbest Bırakmamak

❌ **Yanlış Kullanım:** Arka plan görevlerinde kullanılan kaynakları temizlememek.

```csharp
public class MyBackgroundService : BackgroundService
{
    private readonly Timer _timer = new Timer(OnTimerElapsed);

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _timer.Change(0, 1000); // Timer başlatılır ancak temizlenmez
        return Task.CompletedTask;
    }
}
```

✅ **İdeal Kullanım:** `Dispose` metodunu kullanarak kaynakları temizleyin.

```csharp
public class MyBackgroundService : BackgroundService, IDisposable
{
    private readonly Timer _timer = new Timer(OnTimerElapsed);

    protected override Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _timer.Change(0, 1000);
        return Task.CompletedTask;
    }

    public override void Dispose()
    {
        _timer?.Dispose();
        base.Dispose();
    }
}
```

---

## 6. Performansı İzlememek

❌ **Yanlış Kullanım:** Arka plan görevlerinin performansını ve durumunu izlememek.

✅ **İdeal Kullanım:** İzleme araçları kullanarak performansı ölçün.

- **Örnek Araçlar:** Application Insights, Prometheus, Grafana

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var stopwatch = Stopwatch.StartNew();

    while (!stoppingToken.IsCancellationRequested)
    {
        stopwatch.Restart();
        await DoWorkAsync();
        Console.WriteLine($"Görev süresi: {stopwatch.ElapsedMilliseconds} ms");
    }
}
```