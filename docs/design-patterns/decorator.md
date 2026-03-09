# Decorator Pattern

Decorator deseni, mevcut nesnelere dinamik olarak yeni davranışlar ekler; yanlış kullanımlar ise karmaşık kalıtım hiyerarşilerine ve bakım zorluğuna neden olur.

---

## 1. Kalıtım ile Davranış Ekleme

❌ **Yanlış Kullanım:** Her yeni davranış için alt sınıf türetmek.

```csharp
public class OrderService : BaseOrderService { }
public class LoggingOrderService : OrderService { }
public class CachingLoggingOrderService : LoggingOrderService { }
public class ValidatingCachingLoggingOrderService : CachingLoggingOrderService { }
// Sınıf patlaması
```

✅ **İdeal Kullanım:** Decorator zinciri ile davranışları birleştirin.

```csharp
public interface IOrderService
{
    Task<Order> GetOrderAsync(int id);
}

public class OrderService : IOrderService
{
    public Task<Order> GetOrderAsync(int id) => _repository.GetAsync(id);
}

public class CachingOrderDecorator : IOrderService
{
    private readonly IOrderService _inner;
    private readonly IMemoryCache _cache;

    public CachingOrderDecorator(IOrderService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"order-{id}",
            _ => _inner.GetOrderAsync(id));
    }
}
```

---

## 2. Cross-Cutting Concern'leri Servise Gömmek

❌ **Yanlış Kullanım:** Loglama, cache, validasyon mantığını servis içine yazmak.

```csharp
public class ProductService : IProductService
{
    public async Task<Product> GetAsync(int id)
    {
        _logger.LogInformation("GetAsync çağrıldı: {Id}", id);
        var cached = _cache.Get<Product>(id);
        if (cached != null) return cached;
        var product = await _repository.GetAsync(id);
        _cache.Set(id, product);
        _logger.LogInformation("GetAsync tamamlandı: {Id}", id);
        return product;
    }
}
```

✅ **İdeal Kullanım:** Her concern'ü ayrı decorator olarak uygulayın.

```csharp
public class LoggingProductDecorator : IProductService
{
    private readonly IProductService _inner;
    private readonly ILogger<LoggingProductDecorator> _logger;

    public LoggingProductDecorator(IProductService inner, ILogger<LoggingProductDecorator> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Product> GetAsync(int id)
    {
        _logger.LogInformation("GetAsync çağrıldı: {Id}", id);
        var result = await _inner.GetAsync(id);
        _logger.LogInformation("GetAsync tamamlandı: {Id}", id);
        return result;
    }
}
```

---

## 3. DI ile Decorator Kayıt Sorunu

❌ **Yanlış Kullanım:** Decorator'ı DI'da yanlış kaydetmek.

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IOrderService, CachingOrderDecorator>(); // OrderService'i ezer
```

✅ **İdeal Kullanım:** Scrutor veya manuel kayıt ile decorator zincirini kurun.

```csharp
builder.Services.AddScoped<OrderService>();
builder.Services.AddScoped<IOrderService>(sp =>
{
    var inner = sp.GetRequiredService<OrderService>();
    var cache = sp.GetRequiredService<IMemoryCache>();
    var logger = sp.GetRequiredService<ILogger<LoggingOrderDecorator>>();
    return new LoggingOrderDecorator(new CachingOrderDecorator(inner, cache), logger);
});
```

---

## 4. Decorator Sırasını Yanlış Belirlemek

❌ **Yanlış Kullanım:** Cache decorator'ını logging'den önce koymak.

```csharp
// Loglama cache'den dönen sonuçları görmez
var service = new CachingDecorator(new LoggingDecorator(new ProductService()));
```

✅ **İdeal Kullanım:** Logging en dışta, cache içte olacak şekilde sıralayın.

```csharp
// Logging -> Caching -> Service
var service = new LoggingDecorator(
    new CachingDecorator(
        new ProductService()), logger);
// Her çağrı loglanır, cache hit/miss durumu izlenebilir
```

---

## 5. Generic Decorator Yazmamak

❌ **Yanlış Kullanım:** Her servis için ayrı logging decorator yazmak.

```csharp
public class LoggingOrderDecorator : IOrderService { /* ... */ }
public class LoggingProductDecorator : IProductService { /* ... */ }
public class LoggingUserDecorator : IUserService { /* ... */ }
```

✅ **İdeal Kullanım:** MediatR Pipeline Behavior ile generic decorator uygulayın.

```csharp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        _logger.LogInformation("Handling {Request}", typeof(TRequest).Name);
        var response = await next();
        _logger.LogInformation("Handled {Request}", typeof(TRequest).Name);
        return response;
    }
}
```