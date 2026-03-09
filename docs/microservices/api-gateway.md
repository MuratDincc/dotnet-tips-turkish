# API Gateway

API Gateway, microservice mimarisinde istemci ile servisler arasında tek giriş noktası sağlar; yanlış yapılandırma performans darboğazı ve tek nokta hatalarına yol açar.

---

## 1. Gateway İçinde İş Mantığı

❌ **Yanlış Kullanım:** API Gateway'de iş mantığı yazmak.

```csharp
app.MapGet("/api/orders/{id}", async (int id, HttpClient orderClient, HttpClient customerClient) =>
{
    var order = await orderClient.GetFromJsonAsync<Order>($"api/orders/{id}");
    var customer = await customerClient.GetFromJsonAsync<Customer>($"api/customers/{order.CustomerId}");

    // İş mantığı gateway'de
    if (order.Total > 10000 && !customer.IsPremium)
        return Results.BadRequest("Limit aşıldı");

    order.CustomerName = customer.Name;
    return Results.Ok(order);
});
```

✅ **İdeal Kullanım:** Gateway sadece yönlendirme ve cross-cutting concern yönetsin.

```csharp
// YARP reverse proxy konfigürasyonu
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

app.MapReverseProxy();

// appsettings.json
{
    "ReverseProxy": {
        "Routes": {
            "orders-route": {
                "ClusterId": "orders-cluster",
                "Match": { "Path": "/api/orders/{**catch-all}" }
            }
        },
        "Clusters": {
            "orders-cluster": {
                "Destinations": {
                    "orders-service": { "Address": "https://order-service/" }
                }
            }
        }
    }
}
```

---

## 2. Rate Limiting Yapmamak

❌ **Yanlış Kullanım:** Gateway'de rate limiting olmadan servisleri açık bırakmak.

```csharp
app.MapReverseProxy(); // Sınırsız istek alabilir, downstream servisler ezilir
```

✅ **İdeal Kullanım:** Rate limiting ile servisleri koruyun.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api-limit", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiter.QueueLimit = 10;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

app.MapReverseProxy().RequireRateLimiting("api-limit");
```

---

## 3. Response Aggregation Yapmamak

❌ **Yanlış Kullanım:** İstemciden birden fazla servis çağrısı yaptırmak.

```csharp
// İstemci tarafında
var order = await httpClient.GetAsync("/api/orders/1");
var customer = await httpClient.GetAsync("/api/customers/5");
var shipping = await httpClient.GetAsync("/api/shipping/order/1");
// 3 ayrı HTTP çağrısı, yüksek latency
```

✅ **İdeal Kullanım:** Gateway'de BFF pattern ile aggregation yapın.

```csharp
app.MapGet("/api/bff/order-details/{orderId}", async (
    int orderId,
    IOrderServiceClient orderClient,
    ICustomerServiceClient customerClient,
    IShippingServiceClient shippingClient) =>
{
    var orderTask = orderClient.GetAsync(orderId);
    var shippingTask = shippingClient.GetByOrderAsync(orderId);

    var order = await orderTask;
    var customerTask = customerClient.GetAsync(order.CustomerId);

    await Task.WhenAll(customerTask, shippingTask);

    return Results.Ok(new OrderDetailsResponse
    {
        Order = order,
        Customer = customerTask.Result,
        Shipping = shippingTask.Result
    });
});
```

---

## 4. Health Check Proxy Yapmamak

❌ **Yanlış Kullanım:** Downstream servislerin sağlık durumunu kontrol etmemek.

```csharp
app.MapHealthChecks("/health"); // Sadece gateway'in durumunu kontrol eder
```

✅ **İdeal Kullanım:** Downstream servislerin sağlığını da izleyin.

```csharp
builder.Services.AddHealthChecks()
    .AddUrlGroup(new Uri("https://order-service/health"), name: "order-service",
        failureStatus: HealthStatus.Degraded)
    .AddUrlGroup(new Uri("https://customer-service/health"), name: "customer-service",
        failureStatus: HealthStatus.Degraded)
    .AddUrlGroup(new Uri("https://payment-service/health"), name: "payment-service",
        failureStatus: HealthStatus.Unhealthy);

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

---

## 5. Authentication'ı Her Serviste Tekrarlamak

❌ **Yanlış Kullanım:** Her microservice'de ayrı ayrı JWT doğrulaması yapmak.

```csharp
// Order Service
builder.Services.AddAuthentication().AddJwtBearer(/* config */);
// Customer Service
builder.Services.AddAuthentication().AddJwtBearer(/* aynı config */);
// Payment Service
builder.Services.AddAuthentication().AddJwtBearer(/* aynı config */);
```

✅ **İdeal Kullanım:** Gateway'de merkezi authentication yapın, servislere identity bilgisini header ile iletin.

```csharp
// API Gateway
builder.Services.AddAuthentication().AddJwtBearer(options =>
{
    options.Authority = "https://identity-server/";
    options.Audience = "api-gateway";
});

app.Use(async (context, next) =>
{
    if (context.User.Identity?.IsAuthenticated == true)
    {
        context.Request.Headers["X-User-Id"] = context.User.FindFirst("sub")?.Value;
        context.Request.Headers["X-User-Role"] = context.User.FindFirst("role")?.Value;
    }
    await next();
});

// Downstream servislerde sadece header kontrol edilir
public class UserContext : IUserContext
{
    private readonly IHttpContextAccessor _accessor;
    public string UserId => _accessor.HttpContext?.Request.Headers["X-User-Id"];
    public string Role => _accessor.HttpContext?.Request.Headers["X-User-Role"];
}
```