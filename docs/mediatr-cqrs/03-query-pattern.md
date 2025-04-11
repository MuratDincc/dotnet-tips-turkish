# Query Pattern

Query Pattern, veri okuma işlemlerini temsil eden bir pattern'dır. MediatR'da query'ler de `IRequest<T>` interface'ini implemente eder.

## Query Özellikleri

1. **Read-Only**: Query'ler sadece veri okumalıdır
2. **Tek Sorumluluk**: Her query tek bir veri seti dönmelidir
3. **Performans**: Query'ler optimize edilmiş olmalıdır
4. **Caching**: Mümkünse caching kullanılmalıdır

## Query Örnekleri

### Basit Query
```csharp
public class GetProductByIdQuery : IRequest<ProductDto>
{
    public int Id { get; set; }
}

public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto>
{
    private readonly IProductRepository _repository;

    public GetProductByIdQueryHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<ProductDto> Handle(GetProductByIdQuery request, CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id);
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price
        };
    }
}
```

### Complex Query
```csharp
public class GetProductsQuery : IRequest<PagedList<ProductDto>>
{
    public string SearchTerm { get; set; }
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    public string SortBy { get; set; }
    public bool SortDescending { get; set; }
}

public class GetProductsQueryHandler : IRequestHandler<GetProductsQuery, PagedList<ProductDto>>
{
    private readonly IProductRepository _repository;

    public GetProductsQueryHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<PagedList<ProductDto>> Handle(GetProductsQuery request, CancellationToken cancellationToken)
    {
        var query = _repository.GetAll();

        if (!string.IsNullOrWhiteSpace(request.SearchTerm))
        {
            query = query.Where(p => p.Name.Contains(request.SearchTerm));
        }

        query = request.SortDescending
            ? query.OrderByDescending(p => EF.Property<object>(p, request.SortBy))
            : query.OrderBy(p => EF.Property<object>(p, request.SortBy));

        var totalCount = await query.CountAsync();
        var items = await query
            .Skip((request.PageNumber - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(p => new ProductDto
            {
                Id = p.Id,
                Name = p.Name,
                Price = p.Price
            })
            .ToListAsync();

        return new PagedList<ProductDto>(items, totalCount, request.PageNumber, request.PageSize);
    }
}
```

## Query Best Practices

1. **Query Optimization**
   - Select sadece ihtiyaç duyulan alanları
   - Include'ları dikkatli kullanın
   - Index'leri doğru kullanın

2. **Query Handler Organization**
   - Repository pattern kullanın
   - DTO'ları kullanın
   - Mapping işlemlerini otomatize edin

3. **Caching**
   - Query sonuçlarını cache'leyin
   - Cache invalidation stratejisi belirleyin
   - Distributed cache kullanın

4. **Testing**
   - Query'leri unit test edin
   - Integration testler yazın
   - Performance testleri yapın

## Query Pipeline

Query'ler pipeline üzerinden geçer:

1. Caching
2. Validation
3. Authorization
4. Logging
5. Handler

```csharp
public class QueryPipelineBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ICacheService _cache;

    public QueryPipelineBehavior(ICacheService cache)
    {
        _cache = cache;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var cacheKey = $"{typeof(TRequest).Name}-{request.GetHashCode()}";
        
        var cachedResponse = await _cache.GetAsync<TResponse>(cacheKey);
        if (cachedResponse != null)
        {
            return cachedResponse;
        }

        var response = await next();
        await _cache.SetAsync(cacheKey, response);
        
        return response;
    }
}
``` 