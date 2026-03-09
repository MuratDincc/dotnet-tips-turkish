# REST API Best Practices

RESTful API tasarımı, tutarlı ve kullanılabilir API'ler oluşturmayı sağlar; yanlış tasarım kafa karıştırıcı ve bakımı zor API'lere yol açar.

---

## 1. Fiil Bazlı URL Kullanmak

❌ **Yanlış Kullanım:** URL'de fiil kullanmak.

```csharp
app.MapGet("/api/getProducts", GetAll);
app.MapPost("/api/createProduct", Create);
app.MapPost("/api/deleteProduct/{id}", Delete);
app.MapPost("/api/updateProduct", Update);
```

✅ **İdeal Kullanım:** İsim bazlı URL ve HTTP metod ile eylemi belirtin.

```csharp
var products = app.MapGroup("/api/products");

products.MapGet("/", GetAll);          // GET    /api/products
products.MapGet("/{id}", GetById);     // GET    /api/products/5
products.MapPost("/", Create);          // POST   /api/products
products.MapPut("/{id}", Update);       // PUT    /api/products/5
products.MapDelete("/{id}", Delete);    // DELETE /api/products/5
```

---

## 2. Tutarsız HTTP Status Code Döndürmek

❌ **Yanlış Kullanım:** Her durumda 200 OK döndürmek.

```csharp
app.MapPost("/api/orders", async (CreateOrderDto dto, IOrderService service) =>
{
    try
    {
        var order = await service.CreateAsync(dto);
        return Results.Ok(new { success = true, data = order });
    }
    catch (Exception ex)
    {
        return Results.Ok(new { success = false, error = ex.Message }); // Hata da 200!
    }
});
```

✅ **İdeal Kullanım:** Duruma uygun HTTP status code döndürün.

```csharp
app.MapPost("/api/orders", async (CreateOrderDto dto, IOrderService service) =>
{
    var order = await service.CreateAsync(dto);
    return TypedResults.Created($"/api/orders/{order.Id}", order); // 201 Created
});

app.MapGet("/api/orders/{id}", async (int id, IOrderService service) =>
{
    var order = await service.GetByIdAsync(id);
    return order is not null
        ? TypedResults.Ok(order)           // 200 OK
        : TypedResults.NotFound();          // 404 Not Found
});

app.MapDelete("/api/orders/{id}", async (int id, IOrderService service) =>
{
    await service.DeleteAsync(id);
    return TypedResults.NoContent();        // 204 No Content
});
```

---

## 3. Pagination Yapmamak

❌ **Yanlış Kullanım:** Tüm kayıtları tek seferde döndürmek.

```csharp
app.MapGet("/api/products", async (AppDbContext db) =>
{
    return await db.Products.ToListAsync(); // 100.000 kayıt döner
});
```

✅ **İdeal Kullanım:** Sayfalama ile sınırlı veri döndürün.

```csharp
public record PagedResponse<T>(List<T> Items, int TotalCount, int Page, int PageSize)
{
    public bool HasNextPage => Page * PageSize < TotalCount;
    public bool HasPreviousPage => Page > 1;
}

app.MapGet("/api/products", async (
    int page = 1,
    int pageSize = 20,
    string? search = null,
    IProductService service = default!) =>
{
    pageSize = Math.Min(pageSize, 100); // Maksimum sayfa boyutu
    var result = await service.GetPagedAsync(page, pageSize, search);
    return TypedResults.Ok(result);
});
```

---

## 4. Problem Details Kullanmamak

❌ **Yanlış Kullanım:** Tutarsız hata formatı döndürmek.

```csharp
return Results.BadRequest("Geçersiz istek");
return Results.BadRequest(new { error = "Stok yetersiz" });
return Results.BadRequest(new { message = "Hata", code = 1001 });
// Her endpoint farklı hata formatı
```

✅ **İdeal Kullanım:** RFC 7807 Problem Details standardını kullanın.

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["traceId"] = ctx.HttpContext.TraceIdentifier;
    };
});

// Kullanım
app.MapPost("/api/orders", async (CreateOrderDto dto, IOrderService service) =>
{
    var result = await service.CreateAsync(dto);
    return result.Match(
        success => TypedResults.Created($"/api/orders/{success.Id}", success),
        error => TypedResults.Problem(
            title: "Sipariş oluşturulamadı",
            detail: error.Message,
            statusCode: 422));
});
```

---

## 5. API Versioning Yapmamak

❌ **Yanlış Kullanım:** Breaking change yaparak mevcut istemcileri kırmak.

```csharp
// v1: { "name": "Ali", "surname": "Yılmaz" }
// v2: { "fullName": "Ali Yılmaz" }
// Eski istemciler bozulur
```

✅ **İdeal Kullanım:** URL versioning ile geriye uyumluluk sağlayın.

```csharp
var v1 = app.MapGroup("/api/v1/users");
v1.MapGet("/", async (IUserService service) =>
{
    var users = await service.GetAllAsync();
    return users.Select(u => new { u.Name, u.Surname }); // v1 format
});

var v2 = app.MapGroup("/api/v2/users");
v2.MapGet("/", async (IUserService service) =>
{
    var users = await service.GetAllAsync();
    return users.Select(u => new { FullName = $"{u.Name} {u.Surname}" }); // v2 format
});
```