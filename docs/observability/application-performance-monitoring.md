# Application Performance Monitoring - APM (Uygulama Performans İzleme)

Uygulama Performans İzleme (**APM - Application Performance Monitoring**), yazılım uygulamalarının performansını ve sağlık durumunu izlemek için kullanılan yöntemlerin bütünüdür. APM araçları sayesinde **gecikmeler, hatalar, yavaş sorgular ve işlem darboğazları** tespit edilebilir.

---

## 1. Neden APM Kullanmalıyız?

✔ **Uygulama performansını ölçmek ve iyileştirmek**  
✔ **Gecikmeleri ve darboğazları analiz etmek**  
✔ **Gerçek zamanlı metrikleri ve uyarıları yönetmek**  
✔ **Servisler arası çağrıları izlemek**  
✔ **Veritabanı sorgularını ve HTTP isteklerini takip etmek**  

APM çözümleri, özellikle **mikroservis mimarilerinde** sistemin genel performansını anlamak için kritik öneme sahiptir.

---

## 2. APM İçin Popüler Araçlar

📌 **.NET için Yaygın Kullanılan APM Araçları:**  
- **Application Insights (Azure)**  
- **New Relic**  
- **Datadog APM**  
- **Elastic APM**  
- **Jaeger Tracing**  
- **Zipkin**  
- **Prometheus + Grafana**  

Bu araçlar sayesinde **HTTP istekleri, veritabanı çağrıları, bellek kullanımı ve CPU tüketimi gibi metrikler** detaylı olarak incelenebilir.

---

## 3. .NET Uygulamalarında APM Entegrasyonu

### **1. Azure Application Insights ile APM**

Azure Application Insights, .NET uygulamalarında performans izleme için en güçlü çözümlerden biridir.

📌 **Application Insights için Gerekli Paketleri Yükleyin:**

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

📌 **Application Insights'ı ASP.NET Core Uygulamasına Dahil Etme:**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApplicationInsightsTelemetry("<YOUR_INSTRUMENTATION_KEY>");
```

📌 **Özel Telemetri Logları Gönderme:**

```csharp
using Microsoft.ApplicationInsights;

var telemetry = new TelemetryClient();
telemetry.TrackEvent("UserLoggedIn");
telemetry.TrackMetric("RequestTime", 120);
telemetry.TrackException(new Exception("Hata oluştu!"));
```

✅ **Application Insights ile:**  
- **Gerçek zamanlı metrikler toplanır.**  
- **Hata analizi yapılabilir.**  
- **Kullanıcı etkileşimleri izlenebilir.**  

---

### **2. Prometheus + Grafana ile Performans İzleme**

📌 **Prometheus ve Grafana’yı Docker ile Çalıştırma:**

```bash
docker run -d -p 9090:9090 --name prometheus prom/prometheus
docker run -d -p 3000:3000 --name grafana grafana/grafana
```

📌 **ASP.NET Core Uygulamalarında Prometheus Metriklerini Toplama:**

```bash
dotnet add package prometheus-net.AspNetCore
```

📌 **Startup.cs veya Program.cs içine ekleyin:**

```csharp
app.UseMetricServer();
app.UseHttpMetrics();
```

✅ **Bu sayede:**  
- **Prometheus ile HTTP istekleri ve gecikmeler analiz edilir.**  
- **Grafana ile metrikler görselleştirilir.**  

---

## 4. Dağıtık İzleme (Distributed Tracing) Kullanımı

Uygulamalarınızda **servisler arası çağrıları izlemek** için **OpenTelemetry, Jaeger veya Zipkin** gibi dağıtık izleme sistemleri kullanılabilir.

📌 **OpenTelemetry ile İzleme:**

```csharp
var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddAspNetCoreInstrumentation()
    .AddHttpClientInstrumentation()
    .AddConsoleExporter()
    .Build();
```

📌 **Jaeger Entegrasyonu:**

```csharp
.AddJaegerExporter(options =>
{
    options.AgentHost = "localhost";
    options.AgentPort = 6831;
})
```

✅ **Bu sayede:**  
- **Servisler arası istekler takip edilir.**  
- **Her isteğin gecikme süresi ölçülür.**  
- **Performans darboğazları tespit edilir.**  

---

## 5. APM Kullanarak Performans Testleri Yapma

📌 **BenchmarkDotNet Kullanarak Performans Ölçümü:**

```bash
dotnet add package BenchmarkDotNet
```

📌 **Örnek Benchmark Testi:**

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

public class PerformanceTests
{
    private readonly List<int> numbers = Enumerable.Range(1, 100000).ToList();

    [Benchmark]
    public int SumWithLinq() => numbers.Sum();

    [Benchmark]
    public int SumWithLoop()
    {
        int sum = 0;
        foreach (var num in numbers)
            sum += num;
        return sum;
    }
}

BenchmarkRunner.Run<PerformanceTests>();
```

✅ **Bu test ile:**  
- **Kodunuzun performansını ölçebilirsiniz.**  
- **Linq vs Döngü gibi karşılaştırmalar yapabilirsiniz.**  

---

## 6. APM En İyi Pratikleri

✔ **Uygulamanın CPU, bellek, disk ve ağ kullanımı izlenmeli.**  
✔ **APM araçları ile gecikmeler ve darboğazlar analiz edilmeli.**  
✔ **Kritik noktalar için metrikler toplanmalı ve alarmlar ayarlanmalı.**  
✔ **Loglar merkezi bir sistemde toplanarak hatalar anlık tespit edilmeli.**