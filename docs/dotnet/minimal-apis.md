# Minimal APIs

Minimal APIs, ASP.NET Core ile hızlı ve basit bir şekilde API oluşturmak için sunulan bir özelliktir. Ancak, doğru şekilde kullanılmadığında performans sorunlarına ve kod karmaşıklığına yol açabilir.

---

## 1. Tek Sorumluluk İlkesi

❌ **Yanlış Kullanım:** Tüm iş mantığını bir endpoint'e dahil etmek.

```csharp
app.MapGet("/users", async (HttpContext context) =>
{
    var users = await GetUsersFromDatabaseAsync();
    if (users == null)
    {
        context.Response.StatusCode = 404;
        await context.Response.WriteAsync("Kullanıcı bulunamadı.");
        return;
    }
    await context.Response.WriteAsJsonAsync(users);
});
```

✅ **İdeal Kullanım:** İş mantığını ayrı bir servise taşıyarak kodun okunabilirliğini artırın.

```csharp
app.MapGet("/users", async (IUserService userService) =>
{
    var users = await userService.GetAllUsersAsync();
    return users != null ? Results.Ok(users) : Results.NotFound("Kullanıcı bulunamadı.");
});
```

---

## 2. Bağımlılıkların Doğrudan Yönetimi

❌ **Yanlış Kullanım:** Bağımlılıkların doğrudan örneklerini oluşturmak.

```csharp
app.MapGet("/products", async () =>
{
    using var dbContext = new ProductDbContext();
    var products = await dbContext.Products.ToListAsync();
    return products;
});
```

✅ **İdeal Kullanım:** Bağımlılık injection kullanarak kodu test edilebilir hale getirin.

```csharp
app.MapGet("/products", async (IProductService productService) =>
{
    var products = await productService.GetAllProductsAsync();
    return Results.Ok(products);
});
```

---

## 3. Yanlış HTTP Durum Kodları

❌ **Yanlış Kullanım:** Tüm durumlar için yalnızca 200 (OK) döndürmek.

```csharp
app.MapGet("/orders/{id}", async (int id, IOrderService orderService) =>
{
    var order = await orderService.GetOrderByIdAsync(id);
    return order; // Hatalı çünkü durum kodları belirtilmemiş.
});
```

✅ **İdeal Kullanım:** HTTP durum kodlarını açıkça belirtmek.

```csharp
app.MapGet("/orders/{id}", async (int id, IOrderService orderService) =>
{
    var order = await orderService.GetOrderByIdAsync(id);
    return order != null ? Results.Ok(order) : Results.NotFound("Sipariş bulunamadı.");
});
```

---

## 4. Middleware'in Yanlış Kullanımı

🔴 **Yanlış Kullanım:** Middleware'i endpoint içinde manuel olarak çağırmak.

```csharp
app.MapGet("/middleware-test", async (HttpContext context) =>
{
    // Hatalı çünkü middleware burada elle yönetiliyor.
    if (!context.Request.Headers.ContainsKey("Authorization"))
    {
        context.Response.StatusCode = 401;
        return;
    }
    await context.Response.WriteAsync("Yetkilendirildi.");
});
```

✅ **İdeal Kullanım:** Middleware'i global bir yapı olarak tanımlamak.

```csharp
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("Authorization"))
    {
        context.Response.StatusCode = 401;
        return;
    }
    await next();
});

app.MapGet("/middleware-test", () => "Yetkilendirildi.");
```

---

## 5. Kaynak Yönetimi

🔴 **Yanlış Kullanım:** Kaynakların doğru şekilde serbest bırakılmaması.

```csharp
app.MapGet("/files", async () =>
{
    var fileStream = new FileStream("data.txt", FileMode.Open);
    var content = await new StreamReader(fileStream).ReadToEndAsync();
    return content; // Hatalı çünkü dosya kapanmıyor.
});
```

✅ **İdeal Kullanım:** Kaynak yönetimini `using` ile kontrol edin.

```csharp
app.MapGet("/files", async () =>
{
    using var fileStream = new FileStream("data.txt", FileMode.Open);
    using var reader = new StreamReader(fileStream);
    var content = await reader.ReadToEndAsync();
    return Results.Ok(content);
});
```

---

## 6. Çok Fazla Endpoint Tanımı

🔴 **Yanlış Kullanım:** Her endpoint için benzer mantığın tekrarlanması.

```csharp
app.MapGet("/get-users", async (IUserService userService) => await userService.GetAllUsersAsync());
app.MapGet("/get-orders", async (IOrderService orderService) => await orderService.GetAllOrdersAsync());
app.MapGet("/get-products", async (IProductService productService) => await productService.GetAllProductsAsync());
```

✅ **İdeal Kullanım:** Ortak davranışları bir yapılandırma metodunda gruplandırın.

```csharp
void MapEndpoints(WebApplication app)
{
    app.MapGet("/users", async (IUserService userService) => await userService.GetAllUsersAsync());
    app.MapGet("/orders", async (IOrderService orderService) => await orderService.GetAllOrdersAsync());
    app.MapGet("/products", async (IProductService productService) => await productService.GetAllProductsAsync());
}

MapEndpoints(app);
```