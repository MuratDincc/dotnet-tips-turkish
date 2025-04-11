# Yapılandırılmış Loglama (Structured Logging)

Yapılandırılmış loglama (**Structured Logging**), geleneksel düz metin loglarının aksine, **JSON veya diğer yapılandırılmış formatlarda logların tutulmasını** sağlayarak daha iyi analiz ve aranabilirlik sunar. Merkezi log yönetim sistemleri (**Elasticsearch, Seq, Grafana, Kibana, Splunk**) ile entegre çalışarak logların işlenmesini kolaylaştırır.

---

## 1. Neden Yapılandırılmış Loglama Kullanmalıyız?

✔ **Logların indekslenmesini ve analiz edilmesini kolaylaştırır.**  
✔ **Merkezi log yönetim sistemleriyle daha iyi entegrasyon sağlar.**  
✔ **Kritik olayları (hatalar, kullanıcı işlemleri) filtrelemeyi mümkün kılar.**  
✔ **Dağıtık sistemlerde olayları ilişkilendirerek hata tespitini hızlandırır.**  

Özellikle **mikroservis mimarilerinde**, yapılandırılmış loglama **hata yönetimi ve performans analizleri** için büyük avantaj sağlar.

---

## 2. .NET Uygulamalarında Yapılandırılmış Loglama Kullanımı

### **1. Serilog ile JSON Formatında Loglama**

Serilog, yapılandırılmış loglama için en popüler .NET kütüphanelerinden biridir.

📌 **Gerekli Paketleri Yükleyin:**

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
```

📌 **Serilog'u Yapılandırma:**

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter())
    .WriteTo.File("logs/log.json", rollingInterval: RollingInterval.Day, formatter: new Serilog.Formatting.Json.JsonFormatter())
    .WriteTo.Seq("http://localhost:5341")
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
```

✅ **Bu yapılandırma:**  
- **Console ve JSON dosyalarına logları yazdırır.**  
- **Seq sunucusuna logları göndererek merkezi izleme sağlar.**  

📌 **Örnek JSON Log Çıktısı:**

```json
{
    "Timestamp": "2024-02-12T14:23:45.678Z",
    "Level": "Information",
    "Message": "Kullanıcı giriş yaptı",
    "Properties": {
        "UserId": "12345",
        "IpAddress": "192.168.1.10",
        "Location": "Istanbul, Turkey"
    }
}
```

✅ **Bu yapılandırma sayesinde:**  
- **Loglar merkezi bir sistemde kolayca aranabilir hale gelir.**  
- **İlgili logları belirli filtreler ile analiz etmek mümkün olur.**  

---

### **2. Yapılandırılmış Loglara Correlation ID Dahil Etme**

**Servisler arası istekleri takip edebilmek için loglara Correlation ID eklemek önemlidir.**

📌 **Middleware ile Correlation ID Entegrasyonu:**

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

        context.Items["CorrelationId"] = correlationId;
        await _next(context);
    }
}
```

📌 **Correlation ID’yi Loglara Ekleyin:**

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

✅ **Bu yapılandırma:**  
- **Logların Correlation ID ile ilişkilendirilmesini sağlar.**  
- **Tüm mikroservislerde bir isteğe ait logları kolayca takip edilebilir hale getirir.**  

---

### **3. Yapılandırılmış Logları Elasticsearch ve Kibana ile Görselleştirme**

Yapılandırılmış loglar **Elasticsearch ve Kibana** ile analiz edilebilir.

📌 **Elasticsearch ve Kibana'yı Docker ile Çalıştırma:**

```bash
docker network create elk

docker run -d --name elasticsearch --net elk -p 9200:9200 -e "discovery.type=single-node" elasticsearch:7.10.0

docker run -d --name kibana --net elk -p 5601:5601 kibana:7.10.0
```

📌 **Serilog ile Elasticsearch’e Log Gönderme:**

```csharp
.WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
{
    AutoRegisterTemplate = true,
    IndexFormat = "logs-dotnet-{0:yyyy.MM}"
})
```

✅ **Bu sayede:**  
- **Elasticsearch logları indeksleyerek analiz edilebilir hale getirir.**  
- **Kibana’da logları görselleştirerek detaylı analiz yapılabilir.**  

📌 **Kibana’da Logları Filtreleme:**  

```json
{
    "query": {
        "match": {
            "correlation.id": "12345-abcde"
        }
    }
}
```

✅ **Bu sayede belirli bir Correlation ID’ye sahip loglar analiz edilebilir.**  

---

## 4. Yapılandırılmış Loglama En İyi Pratikleri

✔ **Tüm logları JSON formatında oluşturun.**  
✔ **Loglara Correlation ID ekleyerek istekleri ilişkilendirin.**  
✔ **Logları merkezi bir sunucuya yönlendirin (Elasticsearch, Seq, Prometheus).**  
✔ **Log seviyelerini doğru kullanın (`Debug`, `Info`, `Warning`, `Error`).**  
✔ **Performans kaybını önlemek için asenkron loglama mekanizmaları kullanın.**