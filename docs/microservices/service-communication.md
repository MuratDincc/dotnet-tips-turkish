# Service Communication

Microservice'ler arası iletişim, sistemin dayanıklılığını ve performansını doğrudan etkiler; yanlış iletişim kalıpları sıkı bağımlılık ve cascading failure'lara yol açar.

---

## 1. Senkron Zincirleme Çağrılar

❌ **Yanlış Kullanım:** Servisler arası senkron HTTP zinciri kurmak.

```csharp
public class OrderService
{
    public async Task<OrderResult> CreateOrderAsync(OrderRequest request)
    {
        var customer = await _httpClient.GetFromJsonAsync<Customer>($"api/customers/{request.CustomerId}");
        var inventory = await _httpClient.GetFromJsonAsync<Stock>($"api/inventory/{request.ProductId}");
        var payment = await _httpClient.PostAsJsonAsync("api/payments", new { Amount = request.Total });
        var shipping = await _httpClient.PostAsJsonAsync("api/shipping", new { OrderId = request.Id });
        // Bir servis çökerse tüm zincir başarısız olur
    }
}
```

✅ **İdeal Kullanım:** Asenkron mesajlaşma ile servisleri gevşek bağlayın.

```csharp
public class OrderService
{
    private readonly IMessageBus _messageBus;
    private readonly IOrderRepository _repository;

    public async Task<OrderResult> CreateOrderAsync(OrderRequest request)
    {
        var order = Order.Create(request.CustomerId, request.ProductId, request.Total);
        await _repository.AddAsync(order);

        await _messageBus.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            ProductId = order.ProductId,
            Total = order.Total
        });

        return new OrderResult(order.Id, OrderStatus.Pending);
    }
}
```

---

## 2. HttpClient Yanlış Kullanımı

❌ **Yanlış Kullanım:** Her çağrıda yeni HttpClient oluşturmak.

```csharp
public class CatalogServiceClient
{
    public async Task<Product> GetProductAsync(int id)
    {
        using var client = new HttpClient(); // Socket exhaustion riski
        client.BaseAddress = new Uri("https://catalog-service/");
        return await client.GetFromJsonAsync<Product>($"api/products/{id}");
    }
}
```

✅ **İdeal Kullanım:** Typed HttpClient ile IHttpClientFactory kullanın.

```csharp
builder.Services.AddHttpClient<ICatalogServiceClient, CatalogServiceClient>(client =>
{
    client.BaseAddress = new Uri("https://catalog-service/");
    client.Timeout = TimeSpan.FromSeconds(5);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

public class CatalogServiceClient : ICatalogServiceClient
{
    private readonly HttpClient _client;

    public CatalogServiceClient(HttpClient client) => _client = client;

    public async Task<Product> GetProductAsync(int id)
        => await _client.GetFromJsonAsync<Product>($"api/products/{id}");
}
```

---

## 3. gRPC Yerine Her Yerde REST Kullanmak

❌ **Yanlış Kullanım:** Servisler arası internal iletişimde REST kullanmak.

```csharp
// Her çağrıda JSON serialize/deserialize maliyeti
public async Task<UserProfile> GetUserAsync(int userId)
{
    var response = await _httpClient.GetAsync($"api/users/{userId}");
    var json = await response.Content.ReadAsStringAsync();
    return JsonSerializer.Deserialize<UserProfile>(json);
}
```

✅ **İdeal Kullanım:** Internal servis iletişiminde gRPC kullanın.

```csharp
// user.proto
service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
}

// Client
public class UserServiceClient
{
    private readonly UserService.UserServiceClient _client;

    public UserServiceClient(UserService.UserServiceClient client) => _client = client;

    public async Task<UserResponse> GetUserAsync(int userId)
    {
        return await _client.GetUserAsync(new GetUserRequest { UserId = userId });
    }
}

// DI kaydı
builder.Services.AddGrpcClient<UserService.UserServiceClient>(options =>
{
    options.Address = new Uri("https://user-service:5001");
});
```

---

## 4. Retry Olmadan Servis Çağrısı

❌ **Yanlış Kullanım:** Geçici hatalarda yeniden deneme yapmamak.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var response = await _httpClient.GetAsync($"api/products/{id}");
    response.EnsureSuccessStatusCode(); // İlk hatada exception fırlatır
    return await response.Content.ReadFromJsonAsync<Product>();
}
```

✅ **İdeal Kullanım:** Polly ile retry ve circuit breaker politikaları ekleyin.

```csharp
builder.Services.AddHttpClient<ICatalogService, CatalogService>()
    .AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3,
        retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))))
    .AddTransientHttpErrorPolicy(p => p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

---

## 5. Timeout Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Varsayılan timeout ile uzun süre beklemek.

```csharp
public async Task<OrderSummary> GetSummaryAsync(int orderId)
{
    var order = await _orderClient.GetAsync(orderId);        // 100s default timeout
    var customer = await _customerClient.GetAsync(order.CustomerId); // 100s daha
    // Toplam 200 saniye beklenebilir
}
```

✅ **İdeal Kullanım:** Her servis çağrısına uygun timeout belirleyin.

```csharp
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>(client =>
{
    client.BaseAddress = new Uri("https://order-service/");
    client.Timeout = TimeSpan.FromSeconds(5);
});

public async Task<OrderSummary> GetSummaryAsync(int orderId)
{
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    var orderTask = _orderClient.GetAsync(orderId, cts.Token);
    var customerTask = _customerClient.GetAsync(orderId, cts.Token);

    await Task.WhenAll(orderTask, customerTask); // Paralel çağrı, toplam max 10s
    return new OrderSummary(orderTask.Result, customerTask.Result);
}
```