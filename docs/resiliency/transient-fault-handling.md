# Transient Fault Handling: Yanlış ve İdeal Kullanım

Geçici hatalar (transient faults), ağ bağlantıları veya harici hizmetlerle iletişim sırasında zaman zaman ortaya çıkan kısa süreli sorunlardır. Polly, geçici hataları ele almak için güçlü araçlar sunarak uygulamalarınızın dayanıklılığını artırır.

---

## 1. Transient Fault Nedir?

Geçici hatalar, genellikle yeniden deneme ile çözülebilen kısa süreli sorunlardır. Örneğin:  
- Zaman aşımı hataları  
- Geçici ağ kesintileri  
- Harici hizmetin geçici olarak kullanılamaması  

❌ **Yanlış Kullanım:** Hataları ele almadan işlemleri tekrar denemek.

```csharp
try
{
    await MakeHttpRequestAsync();
}
catch
{
    await MakeHttpRequestAsync(); // Kontrolsüz yeniden deneme
}
```

✅ **İdeal Kullanım:** Polly ile retry politikası uygulayın.

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 2. Retry Politikaları

Geçici hataları ele almak için Polly ile farklı retry stratejileri tanımlayabilirsiniz.

### **Sabit Gecikmeli Retry:**

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(2));

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

### **Artan Gecikmeli Retry:**

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

### **Jitter (Rastgele Gecikme):**

Jitter, sabit gecikmenin neden olduğu yük yoğunluğunu azaltır.

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt =>
    {
        var jitter = new Random().NextDouble();
        return TimeSpan.FromSeconds(retryAttempt + jitter);
    });

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 3. Timeout ve Retry Kombinasyonu

Timeout ve retry politikalarını birleştirerek daha dayanıklı bir çözüm oluşturabilirsiniz.

✅ **Örnek:**

```csharp
var timeoutPolicy = Policy
    .TimeoutAsync(TimeSpan.FromSeconds(10));

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(2));

var combinedPolicy = Policy.WrapAsync(retryPolicy, timeoutPolicy);

await combinedPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```

---

## 4. Performans ve İzleme

- **Loglama:** Retry işlemlerini izlemek için loglar ekleyin.  
- **Metrikler:** Hangi işlemlerin geçici hatalara neden olduğunu belirlemek için metrik toplayın.  

✅ **Örnek:**

```csharp
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt =>
    {
        Console.WriteLine($"Retrying attempt {retryAttempt}...");
        return TimeSpan.FromSeconds(retryAttempt);
    });

await retryPolicy.ExecuteAsync(() => MakeHttpRequestAsync());
```