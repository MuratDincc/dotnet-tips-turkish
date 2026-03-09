# Kibana ile Log Analizi ve Görselleştirme

Kibana, Elasticsearch üzerindeki log verilerini görselleştirir ve analiz eder; yanlış yapılandırma log'ların bulunamamasına ve verimsiz analize yol açar.

---

## 1. Yapılandırılmamış Log Göndermek

❌ **Yanlış Kullanım:** Düz metin logları Elasticsearch'e göndermek.

```csharp
_logger.LogInformation($"Sipariş oluşturuldu: {order.Id}, Müşteri: {order.CustomerId}, Tutar: {order.Total}");
// Düz metin, Kibana'da alan bazlı filtreleme yapılamaz
```

✅ **İdeal Kullanım:** Structured logging ile alan bazlı arama yapın.

```csharp
builder.Host.UseSerilog((context, config) => config
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        IndexFormat = "app-logs-{0:yyyy.MM.dd}",
        AutoRegisterTemplate = true,
        AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv8,
        NumberOfShards = 2,
        NumberOfReplicas = 1
    }));

// Structured log
_logger.LogInformation("Sipariş oluşturuldu: {OrderId}, Müşteri: {CustomerId}, Tutar: {Total}",
    order.Id, order.CustomerId, order.Total);
// Kibana'da OrderId, CustomerId, Total alanları ile filtreleme yapılabilir
```

---

## 2. Index Pattern Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Tek bir index'e tüm logları yazmak.

```csharp
new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
{
    IndexFormat = "application-logs",
    // Tek index sürekli büyür, performans düşer
};
```

✅ **İdeal Kullanım:** Tarih bazlı index rotation uygulayın.

```csharp
new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
{
    IndexFormat = "app-{0:yyyy.MM.dd}",  // Günlük index
    AutoRegisterTemplate = true,
    TemplateName = "app-template",
    BatchPostingLimit = 50,
    Period = TimeSpan.FromSeconds(5)
};
```

```json
// ILM (Index Lifecycle Management) Policy
// PUT _ilm/policy/log-retention
{
  "policy": {
    "phases": {
      "hot": { "actions": { "rollover": { "max_size": "5gb", "max_age": "1d" } } },
      "warm": { "min_age": "7d", "actions": { "shrink": { "number_of_shards": 1 } } },
      "delete": { "min_age": "30d", "actions": { "delete": {} } }
    }
  }
}
```

---

## 3. Dashboard Oluşturmamak

❌ **Yanlış Kullanım:** Kibana'yı sadece log arama için kullanmak.

```
// Kibana Discover sekmesinde her seferinde manuel sorgu yazmak
// Tekrarlayan aramalarda zaman kaybı
```

✅ **İdeal Kullanım:** Önceden tanımlı dashboard'lar oluşturun.

```json
// Kibana saved search örneği - API hataları
{
  "query": {
    "bool": {
      "must": [
        { "range": { "@timestamp": { "gte": "now-1h" } } },
        { "range": { "fields.StatusCode": { "gte": 500 } } }
      ]
    }
  }
}
```

```csharp
// Dashboard için anlamlı log alanları ekleyin
_logger.LogError("API hatası: {StatusCode} {Endpoint} {Method} {Duration}ms {ErrorMessage}",
    context.Response.StatusCode,
    context.Request.Path,
    context.Request.Method,
    stopwatch.ElapsedMilliseconds,
    exception.Message);
```

---

## 4. Log Seviyelerini Yanlış Kullanmak

❌ **Yanlış Kullanım:** Her şeyi Information seviyesinde loglamak.

```csharp
_logger.LogInformation("DB bağlantısı başarısız: {Error}", ex.Message);  // Error olmalı
_logger.LogInformation("Cache miss: {Key}", key);                          // Debug olmalı
_logger.LogInformation("Kullanıcı giriş yaptı: {UserId}", userId);       // Doğru
_logger.LogInformation("Döngü iterasyonu: {i}", i);                       // Trace olmalı
```

✅ **İdeal Kullanım:** Doğru log seviyelerini kullanın.

```csharp
_logger.LogTrace("Döngü iterasyonu: {Index}", i);
_logger.LogDebug("Cache miss, DB'den çekiliyor: {Key}", key);
_logger.LogInformation("Kullanıcı giriş yaptı: {UserId}", userId);
_logger.LogWarning("Rate limit yaklaşıyor: {CurrentRate}/{MaxRate}", current, max);
_logger.LogError(ex, "Sipariş oluşturulamadı: {OrderId}", orderId);
_logger.LogCritical(ex, "Veritabanı bağlantısı tamamen kesildi");

// Ortam bazlı minimum seviye
// appsettings.Production.json
// { "Logging": { "LogLevel": { "Default": "Warning", "MyApp": "Information" } } }
```

---

## 5. Hassas Verileri Loglamak

❌ **Yanlış Kullanım:** PII ve hassas bilgileri loglara yazmak.

```csharp
_logger.LogInformation("Kullanıcı girişi: {Email} {Password}", email, password);
_logger.LogInformation("Ödeme: Kart={CardNumber}", cardNumber);
// KVKK/GDPR ihlali, güvenlik riski
```

✅ **İdeal Kullanım:** Hassas verileri maskeleyin veya hariç tutun.

```csharp
public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    private static readonly HashSet<string> SensitiveFields = new(StringComparer.OrdinalIgnoreCase)
    {
        "password", "token", "secret", "cardnumber", "cvv", "ssn"
    };

    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory,
        out LogEventPropertyValue? result)
    {
        // Hassas alanları maskele
        result = null;
        return false;
    }
}

// Veya Serilog.Expressions ile
builder.Host.UseSerilog((context, config) => config
    .Destructure.ByTransforming<PaymentRequest>(r => new
    {
        r.OrderId,
        CardNumber = "****" + r.CardNumber[^4..],
        CVV = "***"
    }));
```
