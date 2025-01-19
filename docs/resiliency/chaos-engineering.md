# Chaos Engineering

Chaos Engineering, bir sistemin dayanıklılığını test etmek ve hatalara karşı nasıl davrandığını anlamak için tasarlanmış bir disiplindir. Polly ile hata simülasyonu, gecikme enjekte etme ve başarısızlık oranlarını artırma gibi stratejiler uygulanabilir.

---

## 1. Polly ile Ağ Gecikmesi Simülasyonu

Gecikme enjekte ederek sistemi yüksek gecikme koşullarında test edin.

✅ **Örnek: Rastgele Gecikme Ekleme**

```csharp
var latencyPolicy = Policy
    .InjectLatencyAsync(
        enabled: _ => true,
        injectionRate: 0.4, // %40 olasılıkla gecikme
        latency: _ => TimeSpan.FromMilliseconds(new Random().Next(500, 2000))); // 500ms - 2000ms arasında gecikme

await latencyPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Request started...");
    await Task.Delay(100); // Gerçek iş mantığı
    Console.WriteLine("Request completed.");
});
```

---

## 2. Polly ile Hata Enjeksiyonu

Belirli bir olasılıkla hata oluşturun ve sistemin nasıl tepki verdiğini analiz edin.

✅ **Örnek: Simüle Edilmiş Hatalar**

```csharp
var faultPolicy = Policy
    .InjectFaultAsync(
        enabled: _ => true,
        injectionRate: 0.3, // %30 olasılıkla hata
        fault: _ => new Exception("Simulated fault!"));

await faultPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Processing request...");
    await Task.Delay(200); // Gerçek iş mantığı
    Console.WriteLine("Request successfully processed.");
});
```

---

## 3. Polly ile Gecikme ve Hata Kombinasyonu

Gecikme ve hata enjeksiyonunu birleştirerek daha karmaşık senaryolar oluşturabilirsiniz.

✅ **Örnek: Karmaşık Simülasyon**

```csharp
var combinedPolicy = Policy.WrapAsync(
    Policy.InjectLatencyAsync(
        enabled: _ => true,
        injectionRate: 0.3,
        latency: _ => TimeSpan.FromSeconds(2)), // 2 saniye gecikme
    Policy.InjectFaultAsync(
        enabled: _ => true,
        injectionRate: 0.2, // %20 olasılıkla hata
        fault: _ => new Exception("Simulated fault!"))
);

await combinedPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Executing operation...");
    await Task.Delay(150); // Gerçek iş mantığı
    Console.WriteLine("Operation completed.");
});
```

---

## 4. Polly ile Çalışma Yükü Testi

Sistemi yüksek yük altında test etmek için Chaos Engineering stratejileri kullanabilirsiniz.

✅ **Örnek: Toplu İstekler ve Hata Simülasyonu**

```csharp
var bulkPolicy = Policy.InjectFaultAsync(
    enabled: _ => true,
    injectionRate: 0.2,
    fault: _ => new Exception("Random fault injected!"));

var tasks = Enumerable.Range(1, 50).Select(async i =>
{
    try
    {
        await bulkPolicy.ExecuteAsync(async () =>
        {
            Console.WriteLine($"Processing request #{i}");
            await Task.Delay(100);
            Console.WriteLine($"Request #{i} completed.");
        });
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Request #{i} failed: {ex.Message}");
    }
});

await Task.WhenAll(tasks);
```

---

## 5. Polly ile Graceful Degradation (Kademeli Azalma)

Hizmetin tümden çökmesini engellemek için hata durumunda sınırlı bir hizmet sunabilirsiniz.

✅ **Örnek: Hata Durumunda Varsayılan Değer Döndürme**

```csharp
var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "Default value due to failure",
        onFallbackAsync: async (result, context) =>
        {
            Console.WriteLine("Fallback triggered: Returning default value.");
            await Task.CompletedTask;
        });

var result = await fallbackPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Processing request...");
    throw new Exception("Simulated failure!"); // Hata oluştur
});

Console.WriteLine($"Result: {result}");
```

---

## 6. Polly ile Gecikmeli Yanıt ve Retry Politikaları

Kaotik koşullar altında tekrar denemeyi ve gecikmeli yanıtları birleştirin.

✅ **Örnek: Retry + Latency**

```csharp
var retryPolicy = Policy
    .Handle<Exception>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var latencyPolicy = Policy
    .InjectLatencyAsync(
        enabled: _ => true,
        injectionRate: 0.5, // %50 olasılıkla gecikme
        latency: _ => TimeSpan.FromSeconds(1)); // 1 saniye gecikme

var combinedPolicy = Policy.WrapAsync(retryPolicy, latencyPolicy);

await combinedPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Attempting operation...");
    await Task.Delay(200); // Gerçek iş mantığı
    Console.WriteLine("Operation completed.");
});
```