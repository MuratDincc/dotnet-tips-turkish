# Minimal API Endpoint Design

Minimal API'ler, hafif ve hızlı endpoint'ler oluşturur; yanlış tasarım karmaşık ve bakımı zor koda yol açar.

---

## 1. Tüm Endpoint'leri Program.cs'e Yığmak

❌ **Yanlış Kullanım:** Tüm endpoint tanımlarını tek dosyada tutmak.

```csharp
var app = builder.Build();

app.MapGet("/api/products", async (AppDbContext db) => await db.Products.ToListAsync());
app.MapGet("/api/products/{id}", async (int id, AppDbContext db) => await db.Products.FindAsync(id));
app.MapPost("/api/products", async (Product product, AppDbContext db) => { /* ... */ });
app.MapGet("/api/orders", async (AppDbContext db) => await db.Orders.ToListAsync());
app.MapPost("/api/orders", async (Order order, AppDbContext db) => { /* ... */ });
// 100+ endpoint tek dosyada
```

✅ **İdeal Kullanım:** Route group extension method'ları ile organize edin.

```csharp
// Program.cs
app.MapProductEndpoints();
app.MapOrderEndpoints();

// ProductEndpoints.cs
public static class ProductEndpoints
{
    public static void MapProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/products").WithTags("Products");

        group.MapGet("/", GetAll);
        group.MapGet("/{id}", GetById);
        group.MapPost("/", Create);
    }

    private static async Task<Ok<List<ProductDto>>> GetAll(IProductService service)
    {
        var products = await service.GetAllAsync();
        return TypedResults.Ok(products);
    }

    private static async Task<Results<Ok<ProductDto>, NotFound>> GetById(int id, IProductService service)
    {
        var product = await service.GetByIdAsync(id);
        return product is not null ? TypedResults.Ok(product) : TypedResults.NotFound();
    }
}
```

---

## 2. Typed Results Kullanmamak

❌ **Yanlış Kullanım:** IResult ile belirsiz dönüş tipi.

```csharp
app.MapGet("/api/products/{id}", async (int id, AppDbContext db) =>
{
    var product = await db.Products.FindAsync(id);
    if (product == null) return Results.NotFound();
    return Results.Ok(product);
}); // OpenAPI dökümantasyonu dönüş tipini bilemez
```

✅ **İdeal Kullanım:** TypedResults ile OpenAPI uyumlu endpoint yazın.

```csharp
app.MapGet("/api/products/{id}", async Task<Results<Ok<ProductDto>, NotFound>> (int id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is not null
        ? TypedResults.Ok(product)
        : TypedResults.NotFound();
}).WithName("GetProductById")
  .WithOpenApi();
```

---

## 3. Endpoint Filter Kullanmamak

❌ **Yanlış Kullanım:** Her endpoint'te validasyon tekrarlamak.

```csharp
app.MapPost("/api/products", async (CreateProductDto dto, AppDbContext db) =>
{
    if (string.IsNullOrEmpty(dto.Name)) return Results.BadRequest("Ad boş olamaz");
    if (dto.Price <= 0) return Results.BadRequest("Fiyat pozitif olmalı");
    // ...
});
```

✅ **İdeal Kullanım:** Endpoint filter ile validasyonu merkezileştirin.

```csharp
public class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is null) return await next(context);

        var argument = context.Arguments.OfType<T>().FirstOrDefault();
        if (argument is null) return await next(context);

        var result = await validator.ValidateAsync(argument);
        if (!result.IsValid)
            return TypedResults.ValidationProblem(result.ToDictionary());

        return await next(context);
    }
}

app.MapPost("/api/products", Create)
    .AddEndpointFilter<ValidationFilter<CreateProductDto>>();
```

---

## 4. Route Group Özelliklerini Kullanmamak

❌ **Yanlış Kullanım:** Her endpoint'e ayrı ayrı attribute eklemek.

```csharp
app.MapGet("/api/admin/users", GetUsers).RequireAuthorization("Admin");
app.MapPost("/api/admin/users", CreateUser).RequireAuthorization("Admin");
app.MapDelete("/api/admin/users/{id}", DeleteUser).RequireAuthorization("Admin");
```

✅ **İdeal Kullanım:** Route group ile ortak konfigürasyonu bir kez tanımlayın.

```csharp
var admin = app.MapGroup("/api/admin")
    .RequireAuthorization("Admin")
    .WithTags("Admin");

admin.MapGet("/users", GetUsers);
admin.MapPost("/users", CreateUser);
admin.MapDelete("/users/{id}", DeleteUser);

var publicApi = app.MapGroup("/api/public")
    .AllowAnonymous()
    .WithTags("Public");

publicApi.MapGet("/products", GetProducts);
publicApi.MapGet("/categories", GetCategories);
```

---

## 5. Dependency Injection'ı Yanlış Kullanmak

❌ **Yanlış Kullanım:** HttpContext'ten manuel servis çözümleme.

```csharp
app.MapGet("/api/products", async (HttpContext context) =>
{
    var service = context.RequestServices.GetRequiredService<IProductService>();
    var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
    return await service.GetAllAsync();
});
```

✅ **İdeal Kullanım:** Parametrelerde doğrudan servis injection kullanın.

```csharp
app.MapGet("/api/products", async (
    IProductService service,
    ILogger<Program> logger,
    CancellationToken ct) =>
{
    logger.LogInformation("Ürünler listeleniyor");
    return TypedResults.Ok(await service.GetAllAsync(ct));
});
```