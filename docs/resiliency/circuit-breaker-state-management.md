# Circuit Breaker State Management

Circuit Breaker, bir hizmetin hataları kontrol altına almasını sağlayarak sistemin aşırı yüklenmesini önleyen bir dayanıklılık mekanizmasıdır. Polly ile Circuit Breaker'ın durumlarını yönetmek, daha hassas ve etkili bir hata kontrolü sağlar.

---

## 1. Circuit Breaker Nedir?

Circuit Breaker, sistemin üç ana durumda çalışmasını sağlar:  
- **Closed:** Tüm istekler işlenir ve hata oranı izlenir.  
- **Open:** Hata eşiği aşıldığında tüm istekler reddedilir.  
- **Half-Open:** Kısıtlı sayıda isteğe izin verilir ve sistemin durumuna göre tekrar kapalı duruma geçilebilir.  

---

## 2. Polly ile Circuit Breaker Kullanımı

Polly ile Circuit Breaker mekanizması kurarak sistemi aşırı yüklenmekten koruyabilirsiniz.

✅ **Örnek:** Basit Circuit Breaker

```csharp
var circuitBreakerPolicy = Policy
    .Handle<Exception>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30));

await circuitBreakerPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Processing request...");
    await Task.Delay(100); // İş mantığı
    throw new Exception("Simulated failure!");
});
```

---

## 3. Circuit Breaker Durumlarının Yönetimi

Polly, Circuit Breaker'ın durumlarını izlemek ve olaylara tepki vermek için kullanışlı geri bildirim mekanizmaları sunar.

✅ **Örnek:** Durum Değişikliklerini İzleme

```csharp
var circuitBreakerPolicy = Policy
    .Handle<Exception>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (exception, timespan) =>
        {
            Console.WriteLine($"Circuit opened for {timespan.TotalSeconds} seconds due to: {exception.Message}");
        },
        onReset: () =>
        {
            Console.WriteLine("Circuit reset to Closed state.");
        },
        onHalfOpen: () =>
        {
            Console.WriteLine("Circuit in Half-Open state. Testing system health...");
        });

await circuitBreakerPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Attempting operation...");
    await Task.Delay(100);
    throw new Exception("Simulated error.");
});
```

---

## 4. Half-Open Durumu Yönetimi

Half-Open durumunda belirli bir işlem test edilir. Başarılı sonuçlar, devreyi kapalı duruma getirir.

✅ **Örnek:** Half-Open Testi

```csharp
var circuitBreakerPolicy = Policy
    .Handle<Exception>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 2,
        durationOfBreak: TimeSpan.FromSeconds(20),
        onHalfOpen: () =>
        {
            Console.WriteLine("Half-Open: Testing health with a limited operation.");
        });

await circuitBreakerPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Performing health check...");
    await Task.Delay(200); // Test mantığı
});
```

---

## 5. Circuit Breaker ve Retry Kombinasyonu

Circuit Breaker ve Retry politikalarını birleştirerek daha dayanıklı bir çözüm oluşturabilirsiniz.

✅ **Örnek:** Retry ve Circuit Breaker

```csharp
var retryPolicy = Policy
    .Handle<Exception>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var circuitBreakerPolicy = Policy
    .Handle<Exception>()
    .CircuitBreakerAsync(3, TimeSpan.FromSeconds(30));

var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

await combinedPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Executing combined policy...");
    await Task.Delay(150);
    throw new Exception("Simulated failure.");
});
```