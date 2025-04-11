# Tracing ve Correlation ID Yönetimi

Mikroservis mimarilerinde **Tracing (İzleme) ve Correlation ID** kullanımı, sistemde gerçekleşen isteklerin hangi servisten hangi servise geçtiğini anlamak için kritik bir öneme sahiptir. Bir isteğin tüm süreç boyunca izlenebilmesi ve hata tespitinin kolaylaşması için **Correlation ID** kullanımı bir zorunluluktur.

---

## 1. Correlation ID Nedir?

**Correlation ID**, bir istemciden gelen isteğe bağlı olarak oluşturulan ve isteğin tüm yaşam döngüsü boyunca taşınan **benzersiz bir kimliktir**. Tüm mikroservislerde bu ID taşınarak logların ve tracing verilerinin ilişkilendirilmesi sağlanır.

✅ **Correlation ID'nin Sağladıkları:**  
- **Servisler arasında geçen istekleri izleme ve hata ayıklama**  
- **Dağıtık sistemlerde logları birleştirme ve analiz etme**  
- **Performans darboğazlarını tespit etme**  
- **Servis çağrılarını anlamlandırarak istekleri takip etme**  

---

## 2. .NET Uygulamalarında Correlation ID Kullanımı

### **1. Middleware ile Correlation ID Yönetimi**

ASP.NET Core'da bir **middleware** yazarak, her isteğin bir **Correlation ID** taşımasını sağlayabiliriz.

📌 **Middleware Tanımlama:**

```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue("X-Correlation-ID", out var correlationId))
        {
            correlationId = Guid.NewGuid().ToString();
            context.Request.Headers.Append("X-Correlation-ID", correlationId);
        }

        context.Response.Headers.Append("X-Correlation-ID", correlationId);
        context.Items["CorrelationId"] = correlationId;

        await _next(context);
    }
}
```

📌 **Middleware'i Uygulamaya Dahil Etme:**

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseMiddleware<CorrelationIdMiddleware>();
```

✅ **Bu middleware:**  
- Gelen isteklerde **X-Correlation-ID** başlığını kontrol eder.  
- Yoksa **yeni bir Correlation ID oluşturur** ve isteğe ekler.  
- Yanıt başlığına **aynı ID'yi ekleyerek** istemcinin ID'yi almasını sağlar.  

---

### **2. Serilog ile Correlation ID Kullanımı**

Serilog gibi bir loglama framework'ü ile Correlation ID'yi **loglara otomatik olarak dahil edebiliriz**.

📌 **Serilog Konfigürasyonu:**

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Console(outputTemplate: "{Timestamp:HH:mm:ss} [{Level:u3}] {CorrelationId} {Message:lj}{NewLine}{Exception}")
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
```

📌 **Correlation ID'nin Loglara Eklenmesi:**

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task Invoke(HttpContext context, ILogger<LoggingMiddleware> logger)
    {
        var correlationId = context.Items["CorrelationId"]?.ToString() ?? Guid.NewGuid().ToString();
        using (logger.BeginScope(new Dictionary<string, object> { { "CorrelationId", correlationId } }))
        {
            await _next(context);
        }
    }
}
```

📌 **Middleware'i Kullanıma Dahil Etme:**

```csharp
app.UseMiddleware<LoggingMiddleware>();
```

✅ **Bu yapılandırma:**  
- **Tüm loglarda Correlation ID'nin görünmesini** sağlar.  
- **Servisler arası çağrılar bağlamını kaybetmeden takip edilir.**  

---

## 3. OpenTelemetry ile Correlation ID Kullanımı

Eğer dağıtık sistemler için **OpenTelemetry** kullanıyorsanız, Correlation ID'yi tracing verilerine dahil edebilirsiniz.

📌 **OpenTelemetry Konfigürasyonu:**

```csharp
var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddAspNetCoreInstrumentation()
    .AddHttpClientInstrumentation()
    .AddConsoleExporter()
    .Build();
```

📌 **Correlation ID'nin Span İçinde Kullanımı:**

```csharp
using System.Diagnostics;

var activitySource = new ActivitySource("MyApp.Tracing");

using (var activity = activitySource.StartActivity("ProcessOrder"))
{
    activity?.SetTag("correlation.id", "12345-abcde");
}
```

✅ **Bu sayede:**  
- **Tüm tracing span’lerine Correlation ID dahil edilir.**  
- **Jaeger, Zipkin gibi araçlarla ilişkilendirme sağlanır.**  

---

## 4. Dağıtık Loglama İçin Correlation ID Kullanımı

Mikroservislerde **Kibana, Elasticsearch, Grafana** gibi araçlar kullanarak logları tek merkezde toplayabilirsiniz.

📌 **Kibana'da Correlation ID ile Log Arama:**  

```json
{
    "query": {
        "match": {
            "correlation.id": "12345-abcde"
        }
    }
}
```

📌 **Grafana'da Correlation ID Filtreleme:**  

```yaml
datasources:
  - name: logs
    type: elasticsearch
    url: http://elasticsearch:9200
    jsonData:
      index: "logs-*"
```

✅ **Bu sayede:**  
- **Mikroservisler arası istekler tek bir panelden izlenebilir.**  
- **Belirli bir isteğe ait tüm loglar filtrelenerek analiz edilebilir.**  

---

## 5. Correlation ID En İyi Pratikleri

✔ **Tüm HTTP isteklerinde `X-Correlation-ID` başlığı kullanılmalıdır.**  
✔ **Eğer istemciden gelmiyorsa, API kendisi oluşturmalıdır.**  
✔ **Servisler arası çağrılarda Correlation ID taşınmalıdır.**  
✔ **Loglar merkezi bir sistemde (ELK, Grafana, Seq) saklanmalı ve aranabilir olmalıdır.**  
✔ **Tracing ve monitoring araçları ile Correlation ID ilişkilendirilmelidir.**