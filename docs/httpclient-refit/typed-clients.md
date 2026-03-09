# HttpClient ve Refit

HttpClient yönetimi ve Refit ile deklaratif HTTP istemcileri oluşturmak API entegrasyonlarını kolaylaştırır; yanlış kullanımlar socket tükenmesine ve bakım zorluğuna yol açar.

---

## 1. Her Çağrıda HttpClient Oluşturmak

❌ **Yanlış Kullanım:** `new HttpClient()` ile socket exhaustion riski.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    using var client = new HttpClient(); // Her çağrıda yeni socket
    client.BaseAddress = new Uri("https://api.example.com");
    return await client.GetFromJsonAsync<Product>($"/products/{id}");
}
// Socket'lar TIME_WAIT durumunda birikir, port tükenir
```

✅ **İdeal Kullanım:** IHttpClientFactory ile typed client kullanın.

```csharp
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

public class ProductApiClient : IProductApiClient
{
    private readonly HttpClient _client;

    public ProductApiClient(HttpClient client) => _client = client;

    public async Task<Product> GetProductAsync(int id)
        => await _client.GetFromJsonAsync<Product>($"/products/{id}");
}
```

---

## 2. Manuel HTTP Çağrıları Yazmak

❌ **Yanlış Kullanım:** Her endpoint için boilerplate kod tekrarlamak.

```csharp
public async Task<List<Product>> GetAllAsync()
{
    var response = await _client.GetAsync("/products");
    response.EnsureSuccessStatusCode();
    var json = await response.Content.ReadAsStringAsync();
    return JsonSerializer.Deserialize<List<Product>>(json);
}

public async Task<Product> CreateAsync(Product product)
{
    var json = JsonSerializer.Serialize(product);
    var content = new StringContent(json, Encoding.UTF8, "application/json");
    var response = await _client.PostAsync("/products", content);
    response.EnsureSuccessStatusCode();
    return JsonSerializer.Deserialize<Product>(await response.Content.ReadAsStringAsync());
}
```

✅ **İdeal Kullanım:** Refit ile deklaratif API tanımı yapın.

```csharp
public interface IProductApi
{
    [Get("/products")]
    Task<List<Product>> GetAllAsync();

    [Get("/products/{id}")]
    Task<Product> GetByIdAsync(int id);

    [Post("/products")]
    Task<Product> CreateAsync([Body] Product product);

    [Put("/products/{id}")]
    Task<Product> UpdateAsync(int id, [Body] Product product);

    [Delete("/products/{id}")]
    Task DeleteAsync(int id);
}

builder.Services.AddRefitClient<IProductApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.example.com"));
```

---

## 3. Delegating Handler Kullanmamak

❌ **Yanlış Kullanım:** Her istekte manuel header ekleme.

```csharp
public async Task<Product> GetProductAsync(int id)
{
    _client.DefaultRequestHeaders.Authorization =
        new AuthenticationHeaderValue("Bearer", await GetTokenAsync());
    _client.DefaultRequestHeaders.Add("X-Correlation-Id", Guid.NewGuid().ToString());
    return await _client.GetFromJsonAsync<Product>($"/products/{id}");
}
```

✅ **İdeal Kullanım:** Delegating handler ile cross-cutting concern'leri yönetin.

```csharp
public class AuthenticationHandler : DelegatingHandler
{
    private readonly ITokenService _tokenService;

    public AuthenticationHandler(ITokenService tokenService) => _tokenService = tokenService;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var token = await _tokenService.GetAccessTokenAsync();
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        request.Headers.Add("X-Correlation-Id", Activity.Current?.Id ?? Guid.NewGuid().ToString());

        return await base.SendAsync(request, ct);
    }
}

builder.Services.AddTransient<AuthenticationHandler>();

builder.Services.AddRefitClient<IProductApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("https://api.example.com"))
    .AddHttpMessageHandler<AuthenticationHandler>();
```

---

## 4. Hata Yönetimi Yapmamak

❌ **Yanlış Kullanım:** HTTP hata kodlarını görmezden gelmek.

```csharp
public async Task<Product?> GetProductAsync(int id)
{
    return await _client.GetFromJsonAsync<Product>($"/products/{id}");
    // 404, 500 vb. durumlar exception fırlatır, kullanıcıya anlamsız hata döner
}
```

✅ **İdeal Kullanım:** HTTP status code'larına göre yapılandırılmış hata yönetimi yapın.

```csharp
public async Task<Result<Product>> GetProductAsync(int id)
{
    var response = await _client.GetAsync($"/products/{id}");

    return response.StatusCode switch
    {
        HttpStatusCode.OK => Result<Product>.Success(
            await response.Content.ReadFromJsonAsync<Product>()),
        HttpStatusCode.NotFound => Result<Product>.Failure("Ürün bulunamadı"),
        HttpStatusCode.TooManyRequests => Result<Product>.Failure("Rate limit aşıldı"),
        _ => Result<Product>.Failure($"API hatası: {response.StatusCode}")
    };
}

// Refit ile hata yönetimi
builder.Services.AddRefitClient<IProductApi>(new RefitSettings
{
    ExceptionFactory = async response =>
    {
        if (response.IsSuccessStatusCode) return null;
        var content = await response.Content.ReadAsStringAsync();
        return new ApiException(response.RequestMessage, response.StatusCode, content);
    }
});
```

---

## 5. Resilience Eklememek

❌ **Yanlış Kullanım:** Geçici hatalarda yeniden deneme yapmamak.

```csharp
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>();
// Retry, circuit breaker, timeout yok
```

✅ **İdeal Kullanım:** Microsoft.Extensions.Http.Resilience ile resilience ekleyin.

```csharp
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.AddStandardResilienceHandler(options =>
{
    options.Retry.MaxRetryAttempts = 3;
    options.Retry.Delay = TimeSpan.FromSeconds(1);
    options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);
    options.AttemptTimeout.Timeout = TimeSpan.FromSeconds(5);
    options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
});
```