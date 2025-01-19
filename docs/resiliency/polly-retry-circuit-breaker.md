# Polly ile Retry ve Circuit Breaker

Polly, dayanıklı ve güvenilir uygulamalar geliştirmek için kullanılan güçlü bir .NET kütüphanesidir. Ağ hatalarına, geçici sorunlara veya zaman aşımı hatalarına karşı stratejiler sunarak uygulamanızın daha kararlı çalışmasını sağlar.

---

## 1. Retry (Tekrar Deneme) Politikası

Retry politikası, belirli hata türlerinde işlemi tekrar denemeye olanak tanır.

❌ **Yanlış Kullanım:** Belirli bir politika tanımlamadan tekrar denemek.

```csharp
for (int i = 0; i < 3; i++)
{
    try
    {
        await MakeHttpRequestAsync();
        break;
    }
    catch
    {
        if (i == 2) throw;
    }
}
```

✅ **İdeal Kullanım:** Polly ile tekrar deneme stratejisi tanımlayın.

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 2. Circuit Breaker (Devre Kesici) Politikası

Circuit Breaker, bir işlemde belirli sayıda hata meydana geldikten sonra geçici olarak işlemleri durdurur.

❌ **Yanlış Kullanım:** Hataları kontrolsüz bir şekilde biriktirmek.

```csharp
int failureCount = 0;

try
{
    await MakeHttpRequestAsync();
}
catch
{
    failureCount++;
    if (failureCount > 5)
    {
        throw new Exception("Too many failures!");
    }
}
```

✅ **İdeal Kullanım:** Polly ile Circuit Breaker stratejisi tanımlayın.

```csharp
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(5, TimeSpan.FromMinutes(1));

await circuitBreakerPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 3. Retry ve Circuit Breaker Kombinasyonu

Polly, birden fazla stratejiyi birleştirerek daha esnek politikalar oluşturmanıza olanak tanır.

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(2, TimeSpan.FromSeconds(30));

var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

await combinedPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 4. Performans ve İzleme

- **Loglama:** Polly politikalarını kullanırken loglama yaparak hata ve başarı durumlarını izleyin.
- **Metrics:** Başarısızlık oranlarını ve devre kesici durumlarını ölçmek için metrikler ekleyin.

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt =>
    {
        Console.WriteLine($"Retry attempt: {retryAttempt}");
        return TimeSpan.FromSeconds(retryAttempt);
    });
```