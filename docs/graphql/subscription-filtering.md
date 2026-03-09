# Subscription ve Filtering

GraphQL subscription'lar gerçek zamanlı veri akışı sağlar, filtering ise istemciye esnek sorgulama imkanı verir; yanlış kullanımlar bellek sızıntılarına ve performans sorunlarına yol açar.

---

## 1. Subscription'da Topic Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Tüm event'leri tüm istemcilere göndermek.

```csharp
public class Subscription
{
    [Subscribe]
    public OrderDto OnOrderCreated([EventMessage] OrderDto order) => order;
    // Tüm kullanıcılar tüm siparişleri alır
}
```

✅ **İdeal Kullanım:** Topic ile filtrelenmiş subscription yapın.

```csharp
public class Subscription
{
    [Subscribe]
    [Topic("{userId}_orders")]
    public OrderDto OnMyOrderCreated(
        [EventMessage] OrderDto order,
        int userId) => order;
}

// Event gönderimi
public class OrderService
{
    private readonly ITopicEventSender _sender;

    public async Task<OrderDto> CreateAsync(CreateOrderInput input)
    {
        var order = await SaveOrderAsync(input);
        await _sender.SendAsync($"{order.CustomerId}_orders", order);
        return order;
    }
}
```

---

## 2. Filtering'i Manuel Yazmak

❌ **Yanlış Kullanım:** Her alan için manuel filtre parametresi eklemek.

```csharp
public class Query
{
    public async Task<List<ProductDto>> GetProducts(
        string? name, decimal? minPrice, decimal? maxPrice,
        int? categoryId, bool? isActive,
        [Service] AppDbContext db)
    {
        var query = db.Products.AsQueryable();
        if (name != null) query = query.Where(p => p.Name.Contains(name));
        if (minPrice != null) query = query.Where(p => p.Price >= minPrice);
        // ... her alan için ayrı kontrol
        return await query.Select(p => new ProductDto(p)).ToListAsync();
    }
}
```

✅ **İdeal Kullanım:** HotChocolate filtering middleware kullanın.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddFiltering()
    .AddSorting()
    .AddProjections();

public class Query
{
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<ProductDto> GetProducts([Service] AppDbContext db)
        => db.Products.Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category.Name
        });
}

// İstemci sorgusu:
// {
//   products(where: { price: { gte: 100 }, categoryName: { eq: "Elektronik" } },
//            order: { price: DESC }) {
//     name, price, categoryName
//   }
// }
```

---

## 3. Pagination Kullanmamak

❌ **Yanlış Kullanım:** Tüm listeyi döndürmek.

```csharp
public class Query
{
    public async Task<List<ProductDto>> GetProducts([Service] AppDbContext db)
        => await db.Products.Select(p => new ProductDto(p)).ToListAsync();
    // 100.000 kayıt tek seferde döner
}
```

✅ **İdeal Kullanım:** Cursor-based pagination ile sınırlı veri döndürün.

```csharp
public class Query
{
    [UsePaging(IncludeTotalCount = true, MaxPageSize = 50)]
    [UseFiltering]
    [UseSorting]
    public IQueryable<ProductDto> GetProducts([Service] AppDbContext db)
        => db.Products.Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        });
}

// İstemci sorgusu:
// {
//   products(first: 10, after: "cursor123") {
//     nodes { id, name, price }
//     pageInfo { hasNextPage, endCursor }
//     totalCount
//   }
// }
```

---

## 4. Subscription Bağlantı Yönetimini İhmal Etmek

❌ **Yanlış Kullanım:** WebSocket bağlantı sınırı ve timeout ayarı yapmamak.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddSubscriptionType<Subscription>();

app.UseWebSockets();
// Varsayılan ayarlar, bağlantı limiti yok
```

✅ **İdeal Kullanım:** WebSocket ve subscription ayarlarını yapılandırın.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddSubscriptionType<Subscription>()
    .AddInMemorySubscriptions();

app.UseWebSockets(new WebSocketOptions
{
    KeepAliveInterval = TimeSpan.FromSeconds(30)
});

// Redis ile ölçekleme (birden fazla sunucu)
builder.Services
    .AddGraphQLServer()
    .AddSubscriptionType<Subscription>()
    .AddRedisSubscriptions(sp =>
        sp.GetRequiredService<IConnectionMultiplexer>());
```

---

## 5. Field Middleware Sırasını Yanlış Yapmak

❌ **Yanlış Kullanım:** Middleware sırasını karıştırmak.

```csharp
public class Query
{
    [UseFiltering]    // Yanlış sıra
    [UsePaging]       // Paging filtering'den önce gelmeli
    [UseProjection]   // Projection en son çalışmalı
    public IQueryable<Product> GetProducts([Service] AppDbContext db)
        => db.Products;
}
```

✅ **İdeal Kullanım:** Middleware'leri doğru sırada uygulayın.

```csharp
public class Query
{
    [UsePaging(IncludeTotalCount = true)]  // 1. Paging (en dış)
    [UseProjection]                         // 2. Projection
    [UseFiltering]                          // 3. Filtering
    [UseSorting]                            // 4. Sorting (en iç)
    public IQueryable<ProductDto> GetProducts([Service] AppDbContext db)
        => db.Products.Select(p => new ProductDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price
        });
}
// Sıra: Sorting → Filtering → Projection → Paging
// Attribute'lar ters sırada yazılır (en dıştaki en üstte)
```
