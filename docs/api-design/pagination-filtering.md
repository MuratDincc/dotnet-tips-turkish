# Pagination, Filtering ve Sorting

Büyük veri setlerinde sayfalama, filtreleme ve sıralama API performansının temelidir; yanlış uygulamalar yavaş response'lara ve aşırı bellek tüketimine yol açar.

---

## 1. Offset Pagination'da Büyük Offset

❌ **Yanlış Kullanım:** Büyük sayfalarda offset ile veritabanı performansını düşürmek.

```csharp
app.MapGet("/api/products", async (int page, int size, AppDbContext db) =>
{
    return await db.Products
        .Skip((page - 1) * size)  // page=5000, size=20 → 100.000 satır skip
        .Take(size)
        .ToListAsync();
});
```

✅ **İdeal Kullanım:** Cursor-based pagination ile performanslı sayfalama yapın.

```csharp
app.MapGet("/api/products", async (int? cursor, int size, AppDbContext db) =>
{
    size = Math.Min(size, 100);

    var query = db.Products.OrderBy(p => p.Id);

    if (cursor.HasValue)
        query = (IOrderedQueryable<Product>)query.Where(p => p.Id > cursor.Value);

    var items = await query.Take(size + 1).ToListAsync();
    var hasMore = items.Count > size;

    if (hasMore) items.RemoveAt(items.Count - 1);

    return TypedResults.Ok(new
    {
        Items = items,
        NextCursor = hasMore ? items.Last().Id : (int?)null,
        HasMore = hasMore
    });
});
```

---

## 2. Filtrelemeyi Ayrı Ayrı Parametrelerle Yapmak

❌ **Yanlış Kullanım:** Her filtre için ayrı parametre eklemek.

```csharp
app.MapGet("/api/products", async (
    string? name, decimal? minPrice, decimal? maxPrice,
    int? categoryId, bool? isActive, string? brand,
    DateTime? createdAfter, DateTime? createdBefore,
    // ... parametre sayısı artıyor
    AppDbContext db) => { /* ... */ });
```

✅ **İdeal Kullanım:** Filter DTO ile parametreleri gruplayın.

```csharp
public record ProductFilter
{
    public string? Search { get; init; }
    public decimal? MinPrice { get; init; }
    public decimal? MaxPrice { get; init; }
    public int? CategoryId { get; init; }
    public bool? IsActive { get; init; }
}

app.MapGet("/api/products", async (
    [AsParameters] ProductFilter filter,
    int page = 1,
    int pageSize = 20,
    IProductService service = default!) =>
{
    return TypedResults.Ok(await service.GetFilteredAsync(filter, page, pageSize));
});

public class ProductService
{
    public async Task<PagedResponse<ProductDto>> GetFilteredAsync(ProductFilter filter, int page, int pageSize)
    {
        var query = _context.Products.AsQueryable();

        if (!string.IsNullOrEmpty(filter.Search))
            query = query.Where(p => p.Name.Contains(filter.Search));
        if (filter.MinPrice.HasValue)
            query = query.Where(p => p.Price >= filter.MinPrice);
        if (filter.MaxPrice.HasValue)
            query = query.Where(p => p.Price <= filter.MaxPrice);
        if (filter.CategoryId.HasValue)
            query = query.Where(p => p.CategoryId == filter.CategoryId);

        var totalCount = await query.CountAsync();
        var items = await query.Skip((page - 1) * pageSize).Take(pageSize).ToListAsync();

        return new PagedResponse<ProductDto>(items.Select(MapToDto).ToList(), totalCount, page, pageSize);
    }
}
```

---

## 3. Sorting İçin Güvensiz String Kullanmak

❌ **Yanlış Kullanım:** Kullanıcı girdisini doğrudan sıralama ifadesi olarak kullanmak.

```csharp
app.MapGet("/api/products", async (string sortBy, AppDbContext db) =>
{
    return await db.Products
        .OrderBy(p => EF.Property<object>(p, sortBy)) // SQL Injection riski, geçersiz kolon
        .ToListAsync();
});
```

✅ **İdeal Kullanım:** Whitelist ile güvenli sıralama yapın.

```csharp
public static class SortHelper
{
    private static readonly Dictionary<string, Expression<Func<Product, object>>> SortColumns = new()
    {
        ["name"] = p => p.Name,
        ["price"] = p => p.Price,
        ["created"] = p => p.CreatedAt
    };

    public static IQueryable<Product> ApplySort(IQueryable<Product> query, string? sortBy, bool descending)
    {
        if (string.IsNullOrEmpty(sortBy) || !SortColumns.ContainsKey(sortBy.ToLower()))
            return query.OrderByDescending(p => p.CreatedAt);

        var sortExpression = SortColumns[sortBy.ToLower()];
        return descending
            ? query.OrderByDescending(sortExpression)
            : query.OrderBy(sortExpression);
    }
}

app.MapGet("/api/products", async (
    string? sortBy,
    bool desc = false,
    IProductService service = default!) =>
{
    return TypedResults.Ok(await service.GetSortedAsync(sortBy, desc));
});
```

---

## 4. Response Envelope Kullanmamak

❌ **Yanlış Kullanım:** Liste yanıtında metadata döndürmemek.

```csharp
app.MapGet("/api/products", async (AppDbContext db) =>
{
    return await db.Products.Take(20).ToListAsync();
    // Toplam kayıt, sayfa bilgisi, filtre bilgisi yok
});
```

✅ **İdeal Kullanım:** Sayfalama metadata'sı ile zengin response döndürün.

```csharp
public record PagedResponse<T>
{
    public List<T> Items { get; init; }
    public PaginationMeta Pagination { get; init; }
}

public record PaginationMeta
{
    public int CurrentPage { get; init; }
    public int PageSize { get; init; }
    public int TotalPages { get; init; }
    public int TotalCount { get; init; }
    public bool HasPreviousPage => CurrentPage > 1;
    public bool HasNextPage => CurrentPage < TotalPages;
}

// Response örneği:
// {
//   "items": [...],
//   "pagination": {
//     "currentPage": 2,
//     "pageSize": 20,
//     "totalPages": 15,
//     "totalCount": 300,
//     "hasPreviousPage": true,
//     "hasNextPage": true
//   }
// }
```

---

## 5. N+1 Sorgu Sorunu

❌ **Yanlış Kullanım:** Listeleme sorgusunda ilişkili veriyi ayrı ayrı çekmek.

```csharp
var products = await _context.Products.ToListAsync();
foreach (var product in products)
{
    product.Category = await _context.Categories.FindAsync(product.CategoryId); // N+1 sorgu
}
```

✅ **İdeal Kullanım:** Projection ile tek sorguda gerekli veriyi çekin.

```csharp
var products = await _context.Products
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        Price = p.Price,
        CategoryName = p.Category.Name // Tek sorgu, JOIN
    })
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```