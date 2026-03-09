# Distributed Tracing ile Hata Takibi

Distributed tracing, dağıtık sistemlerde isteklerin servisler arası yolculuğunu izler; eksik veya yanlış tracing hata tespitini zorlaştırır ve MTTR'ı artırır.

---

## 1. Trace Context Propagation Yapmamak

❌ **Yanlış Kullanım:** Servisler arası çağrılarda trace context'i taşımamak.

```csharp
public class OrderService
{
    public async Task<Order> CreateAsync(CreateOrderDto dto)
    {
        var order = await SaveOrderAsync(dto);
        await _httpClient.PostAsJsonAsync("/api/inventory/reserve", order);
        await _httpClient.PostAsJsonAsync("/api/payment/charge", order);
        // Her servis bağımsız trace oluşturur, ilişkilendirme yok
    }
}
```

✅ **İdeal Kullanım:** W3C Trace Context ile otomatik propagation sağlayın.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .SetResourceBuilder(ResourceBuilder.CreateDefault()
            .AddService("order-service"))
        .AddOtlpExporter(o => o.Endpoint = new Uri("http://collector:4317")));

// HttpClient otomatik olarak traceparent header'ı ekler
// traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

---

## 2. Span'lara Yeterli Bilgi Eklememek

❌ **Yanlış Kullanım:** Boş span'lar oluşturmak.

```csharp
using var activity = ActivitySource.StartActivity("ProcessOrder");
var result = await ProcessOrderAsync(order);
// Span'da hiçbir detay yok, hata analizi yapılamaz
```

✅ **İdeal Kullanım:** Span'lara tag ve event ekleyin.

```csharp
private static readonly ActivitySource Source = new("OrderService");

public async Task<Order> ProcessOrderAsync(Order order)
{
    using var activity = Source.StartActivity("ProcessOrder", ActivityKind.Internal);
    activity?.SetTag("order.id", order.Id);
    activity?.SetTag("order.total", order.TotalAmount);
    activity?.SetTag("customer.id", order.CustomerId);

    try
    {
        activity?.AddEvent(new ActivityEvent("ValidatingOrder"));
        await ValidateAsync(order);

        activity?.AddEvent(new ActivityEvent("ChargingPayment"));
        await ChargePaymentAsync(order);

        activity?.SetTag("order.status", "completed");
        return order;
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

---

## 3. Sampling Stratejisi Belirlememek

❌ **Yanlış Kullanım:** Tüm istekleri trace etmek.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetSampler(new AlwaysOnSampler()));
// Production'da her istek trace edilir, yüksek maliyet ve performans kaybı
```

✅ **İdeal Kullanım:** Akıllı sampling ile maliyeti kontrol edin.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetSampler(new ParentBasedSampler(new TraceIdRatioBasedSampler(0.1)))
        .AddAspNetCoreInstrumentation(options =>
        {
            options.Filter = context =>
            {
                // Health check ve static file isteklerini hariç tut
                var path = context.Request.Path.Value;
                return !path!.Contains("/health") && !path.Contains("/static");
            };
        }));
```

---

## 4. Baggage Kullanmamak

❌ **Yanlış Kullanım:** İş bilgilerini her serviste tekrar sorgulamak.

```csharp
// Payment Service - müşteri bilgisi için Order Service'e geri sorgu
public async Task ChargeAsync(int orderId)
{
    var order = await _orderClient.GetOrderAsync(orderId);
    var customer = await _customerClient.GetCustomerAsync(order.CustomerId);
    // Gereksiz ağ çağrısı
}
```

✅ **İdeal Kullanım:** Baggage ile iş bağlamını servisler arası taşıyın.

```csharp
// Order Service - baggage ekle
Baggage.SetBaggage("customer.tier", customer.Tier);
Baggage.SetBaggage("order.priority", order.Priority.ToString());

// Payment Service - baggage oku
var customerTier = Baggage.GetBaggage("customer.tier");
if (customerTier == "Premium")
    await PriorityChargeAsync(orderId);
```

---

## 5. Trace Backend Olmadan Çalışmak

❌ **Yanlış Kullanım:** Trace verilerini sadece konsola yazdırmak.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddConsoleExporter()); // Trace verileri kaybolur, arama/analiz yapılamaz
```

✅ **İdeal Kullanım:** OTLP exporter ile merkezi backend'e gönderin.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("http://otel-collector:4317");
            options.Protocol = OtlpExportProtocol.Grpc;
        }))
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```
