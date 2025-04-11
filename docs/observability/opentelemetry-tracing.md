# OpenTelemetry ile Dağıtık İzleme (Tracing)

Dağıtık sistemlerde gözlemlenebilirlik sağlamak için **OpenTelemetry**, tracing (izleme), metrik ve loglama gibi kritik telemetri verilerini toplamak için kullanılan açık kaynaklı bir standarttır.

---

## 1. OpenTelemetry Nedir?

OpenTelemetry, mikroservisler ve dağıtık sistemlerde uygulama gözlemlenebilirliğini artırmak için geliştirilen bir **Observability Framework**'tür. Tracing, metrik toplama ve loglama desteği sağlar.

✅ **OpenTelemetry’nin Sağladıkları:**  
- Servisler arasında **istek takibi** (Distributed Tracing)  
- **Hata analizi** ve performans ölçümü  
- **Metrik ve log entegrasyonu**  
- **Prometheus, Jaeger, Zipkin, Grafana gibi araçlarla entegrasyon**  

---

## 2. OpenTelemetry ile Tracing Mantığı

Tracing, bir isteğin birden fazla servis ve işlem boyunca nasıl ilerlediğini takip etmeye yarar. OpenTelemetry, **trace, span ve context** gibi kavramlarla çalışır:

- **Trace:** Bir isteğin uygulama boyunca izlenen yaşam döngüsüdür.
- **Span:** Bir işlemin belirli bir bileşeni tarafından gerçekleştirilen bağımsız bir bölümdür.
- **Context:** Spesifik bir isteğin ilişkili olduğu bağlamı tanımlar.

✅ **Örnek Trace Yapısı:**  

```
[Trace: API Request]
    ├── [Span 1: API Gateway]
    ├── [Span 2: Authentication Service]
    ├── [Span 3: Order Service]
    ├── [Span 4: Payment Service]
```

---

## 3. .NET Uygulamalarında OpenTelemetry Kullanımı

OpenTelemetry'yi .NET uygulamalarına entegre etmek için **OpenTelemetry NuGet paketleri** yüklenmelidir.

### **1. Gerekli Bağımlılıkları Yükleyin**

```bash
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Exporter.Console
dotnet add package OpenTelemetry.Exporter.Zipkin
dotnet add package OpenTelemetry.Trace
```

### **2. OpenTelemetry Konfigürasyonu**

Aşağıdaki yapılandırma ile **tracing** işlemleri başlatılabilir.

```csharp
using OpenTelemetry.Trace;
using OpenTelemetry.Resources;

var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("MyApp"))
    .AddSource("MyApp.Tracing")
    .AddConsoleExporter()
    .AddZipkinExporter(options =>
    {
        options.Endpoint = new Uri("http://localhost:9411/api/v2/spans");
    })
    .Build();
```

✅ **Bu yapılandırma:**  
- **MyApp** adlı bir servis için tracing oluşturur.  
- **Console Exporter** ile verileri terminalde gösterir.  
- **Zipkin Exporter** ile verileri Zipkin servisine gönderir.  

---

## 4. OpenTelemetry ile Trace Oluşturma

### **1. Manuel Trace Başlatma**

Tracing işlemi için manuel span ekleyebiliriz:

```csharp
using System.Diagnostics;

var activitySource = new ActivitySource("MyApp.Tracing");

using (var activity = activitySource.StartActivity("ProcessOrder"))
{
    activity?.SetTag("order.id", "1234");
    activity?.SetTag("order.status", "Processing");
}
```

### **2. Otomatik HTTP İsteklerini İzleme**

Eğer uygulamanızda **HttpClient** ile çağrılar yapıyorsanız, OpenTelemetry bu çağrıları otomatik olarak izleyebilir:

```csharp
var tracerProvider = Sdk.CreateTracerProviderBuilder()
    .AddHttpClientInstrumentation()
    .AddAspNetCoreInstrumentation()
    .Build();
```

✅ **Bu sayede:**  
- **HttpClient çağrıları otomatik olarak izlenir.**  
- **ASP.NET Core uygulamalarında tüm istekler otomatik olarak trace edilir.**  

---

## 5. OpenTelemetry ile Jaeger ve Zipkin Entegrasyonu

OpenTelemetry ile **Jaeger ve Zipkin** gibi popüler tracing araçlarına veri gönderebilirsiniz.

### **1. Jaeger Kullanımı**

Jaeger servisini çalıştırmak için Docker kullanabilirsiniz:

```bash
docker run -d --name jaeger   -e COLLECTOR_ZIPKIN_HTTP_PORT=9411   -p 5775:5775/udp   -p 6831:6831/udp   -p 6832:6832/udp   -p 5778:5778   -p 16686:16686   -p 14268:14268   -p 14250:14250   -p 9411:9411   jaegertracing/all-in-one:1.38
```

Jaeger UI’ye tarayıcınızdan erişebilirsiniz: [http://localhost:16686](http://localhost:16686)  

### **2. Zipkin Kullanımı**

Zipkin servisini çalıştırmak için:

```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

✅ **Jaeger ve Zipkin Entegrasyonu**:

```csharp
.AddZipkinExporter(options =>
{
    options.Endpoint = new Uri("http://localhost:9411/api/v2/spans");
})
.AddJaegerExporter(options =>
{
    options.AgentHost = "localhost";
    options.AgentPort = 6831;
})
```

---

## 6. OpenTelemetry ile Trace Görselleştirme

Aşağıdaki görselde bir dağıtık sistemde **OpenTelemetry ile izlenen işlemler** gösterilmektedir.

![OpenTelemetry Tracing](https://opentelemetry.io/img/logos/opentelemetry-logo-nav.png)

---

## 7. OpenTelemetry Tracing Kullanım Senaryoları

✔ **Mikroservis Mimarileri** → Bir isteğin tüm servisler boyunca nasıl ilerlediğini takip etme.  
✔ **API Performans Analizi** → Hangi API endpoint'lerinin ne kadar sürede yanıt verdiğini ölçme.  
✔ **Hata Tespiti** → Yavaş veya hatalı çalışan servisleri bulma ve analiz etme.  
✔ **Veritabanı Sorgu Takibi** → Yavaş SQL sorgularını tespit etme ve iyileştirme.