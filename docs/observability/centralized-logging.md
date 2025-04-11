# Centralized Logging (Merkezi Loglama)

Merkezi loglama, dağıtık sistemlerde uygulama loglarının **tek bir merkezde toplanmasını**, analiz edilmesini ve aranabilir hale getirilmesini sağlayan bir yöntemdir. Mikroservis mimarilerinde, logların farklı servislerden toplanıp merkezi bir noktaya gönderilmesi **hata ayıklama, performans takibi ve güvenlik analizleri** açısından kritik öneme sahiptir.

---

## 1. Neden Merkezi Loglama Kullanmalıyız?

✔ **Tüm servislerden gelen logları tek bir yerden izleme**  
✔ **Dağıtık sistemlerde hata tespiti ve analiz kolaylığı**  
✔ **Gerçek zamanlı monitoring ve alerting entegrasyonu**  
✔ **Güvenlik olaylarını merkezi olarak analiz etme**  

Merkezi loglama sayesinde, servisler arası çağrılar ve hatalar **bağlam kaybolmadan** takip edilebilir.

---

## 2. Merkezi Loglama için Popüler Araçlar

- **Elastic Stack (ELK - Elasticsearch, Logstash, Kibana)**  
- **Grafana Loki**  
- **Fluentd ve Fluent Bit**  
- **Seq ile Yapılandırılmış Loglama**  
- **Application Insights (Azure)**  
- **Amazon CloudWatch (AWS)**  
- **Google Cloud Logging**  

---

## 3. .NET Uygulamalarında Merkezi Loglama Entegrasyonu

### **1. Serilog ile Yapılandırılmış Loglama**

Serilog, merkezi loglama sistemleriyle kolayca entegre edilebilen güçlü bir .NET loglama kütüphanesidir.

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Elasticsearch
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
```

### **2. Serilog Konfigürasyonu**

Aşağıdaki kod bloğu, Serilog’u **Elasticsearch, File ve Seq** gibi merkezi loglama çözümleriyle nasıl kullanacağınızı gösterir:

```csharp
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/app-log.txt", rollingInterval: RollingInterval.Day)
    .WriteTo.Seq("http://localhost:5341")
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://localhost:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = "logs-dotnet-{0:yyyy.MM}"
    })
    .CreateLogger();

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog();
```

✅ **Bu yapılandırma:**  
- **Console** ve **File** ile yerel loglama yapar.  
- **Seq** ile merkezi log görüntüleme sunar.  
- **Elasticsearch** ile logları indeksleyerek Kibana ile analiz edilebilir hale getirir.  

---

## 4. Elasticsearch ve Kibana Kullanarak Merkezi Loglama

ELK (Elasticsearch, Logstash, Kibana) **logları saklamak, indekslemek ve görselleştirmek** için kullanılan güçlü bir araçtır.

### **1. Elasticsearch ve Kibana’yı Docker ile Çalıştırma**

```bash
docker network create elk

docker run -d --name elasticsearch --net elk -p 9200:9200 -e "discovery.type=single-node" elasticsearch:7.10.0

docker run -d --name kibana --net elk -p 5601:5601 kibana:7.10.0
```

Elasticsearch’e gönderilen logları Kibana UI’den inceleyebilirsiniz: [http://localhost:5601](http://localhost:5601)

---

## 5. Merkezi Loglama İçin Yapılandırılmış (Structured) Log Kullanımı

Logların analiz edilebilir olması için yapılandırılmış (JSON formatında) olması önemlidir.

✅ **Örnek Serilog JSON Log Çıktısı:**

```json
{
    "Timestamp": "2024-02-12T10:23:45.678Z",
    "Level": "Information",
    "Message": "Kullanıcı giriş yaptı",
    "Properties": {
        "UserId": "12345",
        "IpAddress": "192.168.1.10",
        "Location": "Istanbul, Turkey"
    }
}
```

Bu format, logların **Elasticsearch veya diğer merkezi sistemler tarafından kolayca aranmasını ve analiz edilmesini sağlar.**

---

## 6. .NET ile OpenTelemetry Kullanarak Loglama

OpenTelemetry ile merkezi loglama yapmak için aşağıdaki konfigürasyon kullanılabilir:

```csharp
var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.AddOpenTelemetry(options =>
    {
        options.AddConsoleExporter();
        options.AddOtlpExporter(opt =>
        {
            opt.Endpoint = new Uri("http://otel-collector:4317");
        });
    });
});
```

✅ **Bu yapılandırma:**  
- **OpenTelemetry Collector** kullanarak merkezi log toplanmasını sağlar.  
- **OTLP (OpenTelemetry Protocol) ile Prometheus, Grafana gibi sistemlere veri gönderebilir.**  

---

## 7. Merkezi Loglama ve Alert Mekanizmaları

**Merkezi loglama sistemleriyle olay bazlı uyarılar oluşturabilirsiniz.**  
**Örneğin:** Kibana ve Prometheus kullanarak hata oranları belirli bir eşik değeri aştığında **Slack veya e-posta bildirimi** gönderebilirsiniz.

✅ **Örnek Prometheus Alerting Kuralları:**

```yaml
groups:
  - name: LogAlerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status="500"}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Hata oranı çok yüksek!"
          description: "Son 5 dakikada hata oranı %10’u aştı!"
```

**Bu alerting mekanizması:**  
- **Eşik değeri belirlenen hata oranlarını algılar.**  
- **Slack, Telegram veya e-posta bildirimleri tetikleyebilir.**  

---

## 8. Merkezi Loglama En İyi Pratikleri

✔ **Log Seviyelerini Doğru Kullanın:** `Debug`, `Info`, `Warning`, `Error`, `Critical`.  
✔ **Kullanıcı ve İstek Bağlamını Koruyun:** `Correlation ID` ile isteği takip edin.  
✔ **Yapılandırılmış (JSON) Log Formatı Kullanın:** Logları analiz edilebilir hale getirin.  
✔ **Log Saklama Süresini Tanımlayın:** **Log Retention Policy** ile gereksiz logları temizleyin.  
✔ **Merkezi Arşivleme Kullanın:** Elasticsearch, Seq veya Grafana Loki gibi çözümlerle logları yönetin.
