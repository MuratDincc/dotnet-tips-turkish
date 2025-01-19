# Fallback Stratejileri: Yanlış ve İdeal Kullanım

Fallback stratejileri, bir işlemin başarısız olması durumunda alternatif bir çözüm sağlamayı amaçlar. Polly, hata durumlarında sistemin doğru bir şekilde çalışmaya devam etmesini sağlayan güçlü fallback mekanizmaları sunar.

---

## 1. Temel Fallback Kullanımı

Fallback stratejisi, belirli bir işlem başarısız olduğunda bir varsayılan değer döndürmeyi sağlar.

❌ **Yanlış Kullanım:** Hataları ele almadan varsayılan değer döndürmek.

```csharp
try
{
    var result = await MakeHttpRequestAsync();
}
catch
{
    var result = "Default Value"; // Kontrolsüz fallback
}
```

✅ **İdeal Kullanım:** Polly ile fallback stratejisi tanımlayın.

```csharp
var fallbackPolicy = Policy<string>
    .Handle<HttpRequestException>()
    .FallbackAsync("Default Value");

var result = await fallbackPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 2. Fallback ve Retry Kombinasyonu

Fallback ve Retry politikalarını birleştirerek daha dayanıklı bir çözüm oluşturabilirsiniz.

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

var fallbackPolicy = Policy<string>
    .Handle<HttpRequestException>()
    .FallbackAsync("Fallback Value");

var combinedPolicy = Policy.WrapAsync(fallbackPolicy, retryPolicy);

var result = await combinedPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 3. Fallback Stratejisi ile Dinamik Yanıtlar

Fallback sırasında dinamik olarak hesaplanan bir yanıt döndürebilirsiniz.

✅ **Örnek:**

```csharp
var fallbackPolicy = Policy<string>
    .Handle<HttpRequestException>()
    .FallbackAsync(
        fallbackAction: async (cancellationToken) =>
        {
            Console.WriteLine("Fallback triggered. Returning default value.");
            return await Task.FromResult("Dynamically Calculated Value");
        });

var result = await fallbackPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 4. Performans ve İzleme

- **Loglama:** Fallback tetiklendiğinde loglama yaparak hangi durumlarda devreye girdiğini izleyin.
- **Metrikler:** Hangi işlemlerin fallback stratejisine ihtiyaç duyduğunu ölçmek için metrikler ekleyin.

✅ **Örnek:**

```csharp
var fallbackPolicy = Policy<string>
    .Handle<HttpRequestException>()
    .FallbackAsync(
        fallbackAction: async (cancellationToken) =>
        {
            Console.WriteLine("Using fallback due to exception.");
            return "Fallback Result";
        },
        onFallbackAsync: async (exception, context) =>
        {
            Console.WriteLine($"Fallback executed: {exception.Exception.Message}");
            await Task.CompletedTask;
        });

var result = await fallbackPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```