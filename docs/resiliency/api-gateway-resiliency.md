# API Gateway ile Resiliency

API Gateway, mikroservis mimarilerinde bir giriş noktası görevi görerek talepleri yönlendirir, güvenliği sağlar ve dayanıklılığı artırır. Resiliency uygulamalarında API Gateway kullanımı, sistemin genel kararlılığını ve ölçeklenebilirliğini artırır.

---

## 1. API Gateway Nedir?

API Gateway, bir istemci ile mikroservisler arasında bir ara katman olarak çalışır.  
- **Fonksiyonlar:** İstek yönlendirme, yük dengeleme, güvenlik kontrolü, hata yönetimi.
- **Avantajlar:** Merkezi hata yönetimi, performans optimizasyonu, güvenlik.

---

## 2. Polly ile API Gateway Resiliency Stratejileri

API Gateway'de Polly kullanarak hata yönetimi ve dayanıklılık mekanizmaları oluşturabilirsiniz.

### **Retry Politikası**

✅ **Örnek:** Polly ile yeniden deneme mekanizması

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

await retryPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Forwarding request to microservice...");
    await ForwardRequestToMicroserviceAsync();
});
```

---

### **Circuit Breaker Kullanımı**

Circuit Breaker, hatalı bir servise yapılan isteklerin sayısını sınırlandırarak sistemi korur.

✅ **Örnek:** Circuit Breaker ile hata yönetimi

```csharp
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30));

await circuitBreakerPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Processing request...");
    await ForwardRequestToMicroserviceAsync();
});
```

---

## 3. Cache Kullanımı ile Performans İyileştirme

API Gateway'de sık kullanılan verileri önbelleğe alarak hem performansı artırabilir hem de mikroservislerin yükünü azaltabilirsiniz.

✅ **Örnek:** Polly ile Cache

```csharp
var cachePolicy = Policy.CacheAsync<string>(
    cacheProvider: new MemoryCacheProvider(new MemoryCache(new MemoryCacheOptions())),
    ttl: TimeSpan.FromMinutes(5));

await cachePolicy.ExecuteAsync(async context =>
{
    return await GetDataFromMicroserviceAsync();
}, new Context("cache-key"));
```

---

## 4. Timeout Yönetimi

API Gateway, mikroservislerden yanıt beklerken belirli bir süre içinde işlemi sonlandırmalıdır.

✅ **Örnek:** Polly ile Timeout Politikası

```csharp
var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(10));

await timeoutPolicy.ExecuteAsync(async () =>
{
    Console.WriteLine("Calling microservice with timeout...");
    await ForwardRequestToMicroserviceAsync();
});
```

---

## 5. Load Balancing ve Rate Limiting

API Gateway, yük dengeleme ve istek sınırlandırma mekanizmalarını etkin bir şekilde kullanmalıdır.

✅ **Örnek:** Rate Limiting

```csharp
services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? "global",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            });
    });
});

app.UseRateLimiter();
```

---

## 6. Hata Yönetimi ve Fallback Stratejileri

Mikroservisler hata verdiğinde API Gateway'in yedek bir yanıt döndürmesi gerekebilir.

✅ **Örnek:** Fallback ile Yedek Yanıt

```csharp
var fallbackPolicy = Policy<string>
    .Handle<Exception>()
    .FallbackAsync(
        fallbackValue: "Service is temporarily unavailable.",
        onFallbackAsync: async (exception, context) =>
        {
            Console.WriteLine($"Fallback triggered: {exception.Message}");
            await Task.CompletedTask;
        });

var result = await fallbackPolicy.ExecuteAsync(async () =>
{
    throw new Exception("Microservice failure!");
});

Console.WriteLine($"API Gateway Response: {result}");
```

---

## 7. İzleme ve Loglama

API Gateway'de tüm istek ve hata durumlarını izleyerek sistemin dayanıklılığını artırabilirsiniz.

✅ **Örnek:** OpenTelemetry ile İzleme

```csharp
using var tracer = new ActivitySource("APIGateway");

using var activity = tracer.StartActivity("ProcessRequest");
activity?.AddTag("Route", "/api/data");
activity?.AddTag("Method", "GET");

await ForwardRequestToMicroserviceAsync();
```