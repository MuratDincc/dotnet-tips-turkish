# Timeout Yönetimi

Timeout, uygulamalarda uzun süreli işlemleri sınırlamak ve kullanıcı deneyimini korumak için önemli bir stratejidir. Polly, işlemlerinizde zaman aşımı kontrolü için etkili bir çözüm sunar.

---

## 1. Temel Timeout Yönetimi

❌ **Yanlış Kullanım:** Timeout kontrolü olmadan uzun süre beklemek.

```csharp
await MakeHttpRequestAsync(); // Potansiyel olarak sonsuza kadar bekleyebilir
```

✅ **İdeal Kullanım:** Polly ile zaman aşımı kontrolü ekleyin.

```csharp
var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(5));

await timeoutPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 2. Timeout ve Retry Kombinasyonu

Timeout ve Retry politikalarını birleştirerek hem işlemi belirli bir süreyle sınırlayabilir hem de hata durumunda tekrar deneme yapabilirsiniz.

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(5));

var combinedPolicy = Policy.WrapAsync(retryPolicy, timeoutPolicy);

await combinedPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 3. CancellationToken ile Timeout Yönetimi

Polly, `CancellationToken` ile entegre çalışarak işlemleri iptal etmenizi sağlar.

✅ **Örnek:**

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));

var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(10));

await timeoutPolicy.ExecuteAsync(
    async token => await MakeHttpRequestAsync(token),
    cts.Token);
```

---

## 4. Performans ve İzleme

- **Loglama:** Timeout durumlarını izlemek için log ekleyin.
- **Uyarlanabilir Süreler:** Farklı işlemler için farklı timeout süreleri belirleyin.

✅ **Örnek:**

```csharp
var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(5), (context, timespan, task) =>
    {
        Console.WriteLine($"Timeout after {timespan.TotalSeconds} seconds.");
        return Task.CompletedTask;
    });

await timeoutPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```