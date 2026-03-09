# Cross-Cutting Concerns

Cross-cutting concern'ler loglama, validasyon ve exception handling gibi tüm katmanları kesen sorumluluklardır; yanlış yönetim kod tekrarına ve bakım zorluğuna yol açar.

---

## 1. Her Katmanda Loglama Tekrarı

❌ **Yanlış Kullanım:** Her servis metodunda manuel loglama yapmak.

```csharp
public class OrderService
{
    public async Task<Order> CreateAsync(CreateOrderDto dto)
    {
        _logger.LogInformation("CreateAsync başladı: {@Dto}", dto);
        try
        {
            var order = new Order(dto.CustomerId);
            await _repository.AddAsync(order);
            _logger.LogInformation("Sipariş oluşturuldu: {OrderId}", order.Id);
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "CreateAsync hatası: {@Dto}", dto);
            throw;
        }
    }
}
```

✅ **İdeal Kullanım:** MediatR Pipeline Behavior ile loglama merkezileştirin.

```csharp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger) => _logger = logger;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {Request}: {@RequestData}", requestName, request);

        var response = await next();

        _logger.LogInformation("Handled {Request}", requestName);
        return response;
    }
}
```

---

## 2. Validasyon Mantığını Dağıtmak

❌ **Yanlış Kullanım:** Validasyonu birden fazla yerde tekrarlamak.

```csharp
// Controller'da
if (string.IsNullOrEmpty(dto.Name)) return BadRequest();

// Serviste
if (string.IsNullOrEmpty(dto.Name)) throw new ValidationException();

// Repository'de
if (string.IsNullOrEmpty(entity.Name)) throw new ArgumentException();
```

✅ **İdeal Kullanım:** Validasyonu tek bir noktada, pipeline'da yapın.

```csharp
public class CreateProductValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Price).GreaterThan(0);
    }
}

public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

---

## 3. Transaction Yönetimini Her Yerde Tekrarlamak

❌ **Yanlış Kullanım:** Her handler'da transaction açıp kapatmak.

```csharp
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    public async Task<int> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        using var transaction = await _context.Database.BeginTransactionAsync(ct);
        try
        {
            // ... iş mantığı
            await _context.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);
            return order.Id;
        }
        catch
        {
            await transaction.RollbackAsync(ct);
            throw;
        }
    }
}
```

✅ **İdeal Kullanım:** Transaction behavior ile merkezileştirin.

```csharp
public class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly AppDbContext _context;

    public TransactionBehavior(AppDbContext context) => _context = context;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var strategy = _context.Database.CreateExecutionStrategy();

        return await strategy.ExecuteAsync(async () =>
        {
            await using var transaction = await _context.Database.BeginTransactionAsync(ct);
            var response = await next();
            await _context.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);
            return response;
        });
    }
}
```

---

## 4. Performance Monitoring'i Manuel Yapmak

❌ **Yanlış Kullanım:** Her metodda Stopwatch kullanmak.

```csharp
public async Task<Order> GetOrderAsync(int id)
{
    var sw = Stopwatch.StartNew();
    var order = await _repository.GetByIdAsync(id);
    sw.Stop();
    _logger.LogInformation("GetOrderAsync süre: {Elapsed}ms", sw.ElapsedMilliseconds);
    return order;
}
```

✅ **İdeal Kullanım:** Performance behavior ile süre ölçümünü merkezileştirin.

```csharp
public class PerformanceBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    private readonly Stopwatch _timer = new();

    public PerformanceBehavior(ILogger<PerformanceBehavior<TRequest, TResponse>> logger) => _logger = logger;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        _timer.Start();
        var response = await next();
        _timer.Stop();

        if (_timer.ElapsedMilliseconds > 500)
        {
            _logger.LogWarning("Yavaş sorgu: {Request} ({Elapsed}ms)",
                typeof(TRequest).Name, _timer.ElapsedMilliseconds);
        }

        return response;
    }
}
```

---

## 5. Caching'i Servislere Gömmek

❌ **Yanlış Kullanım:** Her serviste cache mantığını tekrarlamak.

```csharp
public class ProductService
{
    public async Task<Product> GetByIdAsync(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null) return JsonSerializer.Deserialize<Product>(cached);

        var product = await _repository.GetByIdAsync(id);
        await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
        return product;
    }
}
```

✅ **İdeal Kullanım:** Caching behavior veya decorator ile merkezileştirin.

```csharp
public interface ICacheableQuery
{
    string CacheKey { get; }
    TimeSpan CacheDuration { get; }
}

public record GetProductQuery(int Id) : IRequest<ProductDto>, ICacheableQuery
{
    public string CacheKey => $"product:{Id}";
    public TimeSpan CacheDuration => TimeSpan.FromMinutes(5);
}

public class CachingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IDistributedCache _cache;

    public CachingBehavior(IDistributedCache cache) => _cache = cache;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (request is not ICacheableQuery cacheable) return await next();

        var cached = await _cache.GetStringAsync(cacheable.CacheKey, ct);
        if (cached != null) return JsonSerializer.Deserialize<TResponse>(cached);

        var response = await next();
        await _cache.SetStringAsync(cacheable.CacheKey,
            JsonSerializer.Serialize(response),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = cacheable.CacheDuration }, ct);

        return response;
    }
}
```