# Graceful Degradation

Graceful Degradation, bir sistemin kısmi bir hata durumunda bile hizmet sunmaya devam etmesini sağlayan bir dayanıklılık stratejisidir. Bu strateji, kritik olmayan özelliklerin geçici olarak kapatılmasını veya azaltılmış bir hizmet sunulmasını içerir.

---

## 1. Graceful Degradation Nedir?

- **Amaç:** Kullanıcı deneyimini koruyarak, sistemin tamamen çökmesini önlemek.  
- **Kullanım Alanları:** Yüksek trafiğe sahip web siteleri, mikroservis mimarileri, dağıtık sistemler.  

Örneğin, bir e-ticaret sitesinde ödeme hizmeti geçici olarak kullanılamıyorsa, kullanıcıların ürün incelemesi yapmaya devam etmesi sağlanabilir.

---

## 2. Polly ile Graceful Degradation

Polly'nin **Fallback** politikası, Graceful Degradation stratejisini uygulamak için etkili bir araçtır.

✅ **Örnek:** Fallback ile Alternatif Yanıt Döndürme

```csharp
var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "Reduced functionality: Please try again later.",
        onFallbackAsync: async (exception, context) =>
        {
            Console.WriteLine($"Fallback triggered: {exception.Message}");
            await Task.CompletedTask;
        });

var result = await fallbackPolicy.ExecuteAsync(async () =>
{
    throw new Exception("Primary service unavailable!");
});

Console.WriteLine($"Response: {result}");
```

---

## 3. Özellik Azaltma (Feature Reduction)

Bir sistemin bazı özelliklerini geçici olarak kapatarak hizmet vermeye devam etmesi sağlanabilir.

✅ **Örnek:** Kritik Olmayan Özellikleri Devre Dışı Bırakma

```csharp
if (!IsCriticalFeatureAvailable())
{
    Console.WriteLine("Critical feature unavailable. Displaying limited functionality.");
    DisplayBasicFeatures();
}
else
{
    DisplayFullFeatures();
}
```

---

## 4. Graceful Degradation ile Cache Kullanımı

Geçici bir hata durumunda, önceden önbelleğe alınmış veriler kullanılabilir.

✅ **Örnek:** Cache ile Yedekleme

```csharp
var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackAction: async cancellationToken =>
        {
            Console.WriteLine("Using cached data due to failure.");
            return "Cached Data";
        });

var result = await fallbackPolicy.ExecuteAsync(async () =>
{
    throw new Exception("Primary service failed!");
});

Console.WriteLine($"Result: {result}");
```

---

## 5. Graceful Degradation ve Retry Kombinasyonu

Hataları yönetmek için Retry ve Graceful Degradation stratejilerini birleştirebilirsiniz.

✅ **Örnek:** Retry ve Fallback Kombinasyonu

```csharp
var retryPolicy = Policy
    .Handle<Exception>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "Service is temporarily unavailable.");

var combinedPolicy = Policy.WrapAsync(fallbackPolicy, retryPolicy);

var result = await combinedPolicy.ExecuteAsync(async () =>
{
    throw new Exception("Service failed!");
});

Console.WriteLine($"Response: {result}");
```

---

## 6. Performans ve İzleme

Graceful Degradation sırasında sistem performansını izlemek önemlidir:  
- **Metrikler:** Azaltılmış hizmetlerin ne sıklıkta kullanıldığını izleyin.  
- **Loglama:** Fallback ve diğer stratejilerin ne zaman devreye girdiğini takip edin.  

✅ **Örnek:** Loglama ile İzleme

```csharp
var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "Limited functionality is active.",
        onFallbackAsync: async (exception, context) =>
        {
            Console.WriteLine($"Fallback triggered due to: {exception.Message}");
            await Task.CompletedTask;
        });

await fallbackPolicy.ExecuteAsync(async () =>
{
    throw new Exception("Simulated failure!");
});
```