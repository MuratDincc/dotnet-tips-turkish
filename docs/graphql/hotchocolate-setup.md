# HotChocolate Kurulum ve Query Tasarımı

HotChocolate, .NET için güçlü bir GraphQL sunucusudur; yanlış kurulum ve query tasarımı performans sorunlarına ve güvenlik açıklarına yol açar.

---

## 1. Tüm Entity'leri Doğrudan Expose Etmek

❌ **Yanlış Kullanım:** Veritabanı entity'lerini direkt GraphQL şemasına eklemek.

```csharp
public class Query
{
    [UseDbContext(typeof(AppDbContext))]
    public IQueryable<User> GetUsers([ScopedService] AppDbContext context)
        => context.Users; // Tüm alanlar (password hash dahil) expose olur
}
```

✅ **İdeal Kullanım:** DTO/projection ile sadece gerekli alanları döndürün.

```csharp
public class Query
{
    public async Task<IEnumerable<UserDto>> GetUsers(
        [Service] IUserService service, CancellationToken ct)
        => await service.GetAllAsync(ct);
}

[ObjectType("User")]
public class UserType : ObjectType<UserDto>
{
    protected override void Configure(IObjectTypeDescriptor<UserDto> descriptor)
    {
        descriptor.Field(u => u.Id).Type<NonNullType<IdType>>();
        descriptor.Field(u => u.FullName).Type<NonNullType<StringType>>();
        descriptor.Field(u => u.Email).Type<NonNullType<StringType>>();
        // PasswordHash gibi hassas alanlar yok
    }
}
```

---

## 2. Query Depth Sınırı Koymamak

❌ **Yanlış Kullanım:** Sınırsız derinlikte query'lere izin vermek.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>();
// Saldırgan: { users { orders { products { reviews { user { orders { ... } } } } } } }
// Sonsuz derinlikte sorgu sunucuyu çökertir
```

✅ **İdeal Kullanım:** Query depth ve complexity sınırları ekleyin.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddQueryType<Query>()
    .AddMutationType<Mutation>()
    .AddMaxExecutionDepthRule(5)
    .AddProjections()
    .AddFiltering()
    .AddSorting()
    .SetRequestOptions(_ => new RequestExecutorOptions
    {
        ExecutionTimeout = TimeSpan.FromSeconds(10)
    });
```

---

## 3. Mutation'ları Doğru Yapılandırmamak

❌ **Yanlış Kullanım:** Mutation'da input validation ve hata yönetimi yapmamak.

```csharp
public class Mutation
{
    public async Task<Product> CreateProduct(string name, decimal price,
        [Service] AppDbContext db)
    {
        var product = new Product { Name = name, Price = price };
        db.Products.Add(product);
        await db.SaveChangesAsync();
        return product;
    }
}
```

✅ **İdeal Kullanım:** Input type, payload pattern ve validation kullanın.

```csharp
public record CreateProductInput(string Name, decimal Price, int CategoryId);

public class CreateProductPayload
{
    public ProductDto? Product { get; init; }
    public IReadOnlyList<UserError>? Errors { get; init; }
}

public record UserError(string Message, string Code);

public class Mutation
{
    public async Task<CreateProductPayload> CreateProduct(
        CreateProductInput input,
        [Service] IProductService service)
    {
        if (string.IsNullOrWhiteSpace(input.Name))
            return new CreateProductPayload
            {
                Errors = new[] { new UserError("Ürün adı boş olamaz", "INVALID_NAME") }
            };

        var product = await service.CreateAsync(input);
        return new CreateProductPayload { Product = product };
    }
}
```

---

## 4. Authorization Eklememek

❌ **Yanlış Kullanım:** Tüm query ve mutation'lara anonim erişim.

```csharp
public class Query
{
    public IQueryable<Order> GetOrders([Service] AppDbContext db)
        => db.Orders; // Herkes tüm siparişleri görebilir
}
```

✅ **İdeal Kullanım:** HotChocolate authorization ile erişim kontrolü yapın.

```csharp
builder.Services
    .AddGraphQLServer()
    .AddAuthorization()
    .AddQueryType<Query>();

public class Query
{
    [Authorize]
    public async Task<IEnumerable<OrderDto>> GetMyOrders(
        [Service] IOrderService service,
        [GlobalState("currentUserId")] int userId,
        CancellationToken ct)
        => await service.GetByUserIdAsync(userId, ct);

    [Authorize(Roles = new[] { "Admin" })]
    public async Task<IEnumerable<OrderDto>> GetAllOrders(
        [Service] IOrderService service, CancellationToken ct)
        => await service.GetAllAsync(ct);
}
```

---

## 5. DataLoader Kullanmamak

❌ **Yanlış Kullanım:** İlişkili veride N+1 sorgu problemi.

```csharp
public class OrderType : ObjectType<OrderDto>
{
    protected override void Configure(IObjectTypeDescriptor<OrderDto> descriptor)
    {
        descriptor.Field("customer")
            .ResolveWith<OrderResolvers>(r => r.GetCustomer(default!, default!));
    }
}

public class OrderResolvers
{
    public async Task<CustomerDto> GetCustomer(
        [Parent] OrderDto order, [Service] AppDbContext db)
        => await db.Customers.FindAsync(order.CustomerId); // Her order için ayrı sorgu
}
```

✅ **İdeal Kullanım:** DataLoader ile batch yükleme yapın.

```csharp
public class CustomerBatchDataLoader : BatchDataLoader<int, CustomerDto>
{
    private readonly IServiceProvider _services;

    public CustomerBatchDataLoader(
        IServiceProvider services,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options) => _services = services;

    protected override async Task<IReadOnlyDictionary<int, CustomerDto>> LoadBatchAsync(
        IReadOnlyList<int> keys, CancellationToken ct)
    {
        await using var scope = _services.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        var customers = await db.Customers
            .Where(c => keys.Contains(c.Id))
            .ToDictionaryAsync(c => c.Id, c => new CustomerDto(c.Id, c.Name), ct);

        return customers;
    }
}

// Kullanım
descriptor.Field("customer")
    .ResolveWith<OrderResolvers>(r => r.GetCustomer(default!, default!));

public async Task<CustomerDto> GetCustomer(
    [Parent] OrderDto order, CustomerBatchDataLoader loader)
    => await loader.LoadAsync(order.CustomerId);
```
