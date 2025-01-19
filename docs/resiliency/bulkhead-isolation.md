# Bulkhead Isolation: Yanlış ve İdeal Kullanım

Bulkhead Isolation, sistemdeki kaynakların belirli bir kısmını ayırarak bir bileşenin arızalanmasının diğer bileşenleri etkilemesini önlemeyi amaçlar. Polly, bu stratejiyi uygulamak için etkili araçlar sunar.

---

## 1. Bulkhead Isolation'ın Önemi

Bulkhead Isolation, bir sistemdeki hataların yayılmasını önler. Örneğin, yoğun bir API çağrısı trafiği diğer işlemlerin performansını düşürmemelidir.

❌ **Yanlış Kullanım:** Kaynakların kontrolsüz şekilde paylaşılması.

```csharp
Parallel.For(0, 100, async i =>
{
    await MakeHttpRequestAsync(); // Tüm çağrılar sınırsız kaynak tüketir
});
```

✅ **İdeal Kullanım:** Polly ile kaynak sınırlarını kontrol altına alın.

```csharp
var bulkheadPolicy = Policy.BulkheadAsync(10, int.MaxValue);

await bulkheadPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 2. Bulkhead ile Kuyruklama

Bulkhead Isolation, aşırı yüklenme durumunda gelen işlemleri bir kuyruğa alabilir.

✅ **Örnek:**

```csharp
var bulkheadPolicy = Policy.BulkheadAsync(
    maxParallelization: 10,
    maxQueuingActions: 20,
    onBulkheadRejectedAsync: context =>
    {
        Console.WriteLine("Request rejected due to bulkhead limit.");
        return Task.CompletedTask;
    });

await bulkheadPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 3. Bulkhead ve Diğer Politikalarla Kombinasyon

Bulkhead Isolation, diğer Polly politikaları ile birleştirilerek daha dayanıklı sistemler oluşturulabilir.

✅ **Örnek:** Retry ve Circuit Breaker ile Bulkhead kullanımı.

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(2, TimeSpan.FromSeconds(30));

var bulkheadPolicy = Policy.BulkheadAsync(10, 20);

var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy, bulkheadPolicy);

await combinedPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 4. Performans ve İzleme

- **Metrik İzleme:** Bulkhead sınırlarına ulaşılıp ulaşılmadığını izleyin.
- **Doğru Parametre Seçimi:** `maxParallelization` ve `maxQueuingActions` değerlerini sistem kapasitesine uygun belirleyin.

✅ **Örnek:**

```csharp
var bulkheadPolicy = Policy.BulkheadAsync(
    maxParallelization: 5,
    maxQueuingActions: 10,
    onBulkheadRejectedAsync: context =>
    {
        Console.WriteLine("Bulkhead limit exceeded. Request rejected.");
        return Task.CompletedTask;
    });

await bulkheadPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```