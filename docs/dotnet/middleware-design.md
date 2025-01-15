# Middleware Tasarımı ve Uygulama

ASP.NET Core'da middleware, bir HTTP isteği ve yanıtı üzerinde işlem yapmak için kullanılan bir yazılım katmanıdır. Middleware'in yanlış tasarlanması veya uygulanması performans sorunlarına ve karmaşık kod yapısına yol açabilir.

---

## 1. Gereksiz Middleware Kullanımı

❌ **Yanlış Kullanım:** Her küçük işlem için ayrı middleware oluşturmak.

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("Request: " + context.Request.Path);
        await _next(context);
    }
}
```

✅ **İdeal Kullanım:** İlgili işlemleri tek bir middleware içinde gruplandırın.

```csharp
public class CombinedMiddleware
{
    private readonly RequestDelegate _next;

    public CombinedMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("Request: " + context.Request.Path);

        if (context.Request.Path.StartsWithSegments("/secure"))
        {
            // Ek güvenlik kontrolü
            if (!context.User.Identity.IsAuthenticated)
            {
                context.Response.StatusCode = 401;
                return;
            }
        }

        await _next(context);
    }
}
```

---

## 2. Middleware'in Yanlış Sıralanması

❌ **Yanlış Kullanım:** Middleware sırasının önemli olduğu durumlarda yanlış sırayla ekleme.

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

✅ **İdeal Kullanım:** Middleware sırasını doğru bir şekilde yapılandırın.

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

> 💡 **Not:** Doğru middleware sırası, uygulamanın düzgün çalışması için hayati önem taşır.

---

## 3. Response'ı Manuel Olarak Yazmak

❌ **Yanlış Kullanım:** Yanıtın tamamını manuel olarak işlemek.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    if (context.Request.Path == "/custom")
    {
        context.Response.StatusCode = 200;
        await context.Response.WriteAsync("Custom Response");
        return;
    }

    await _next(context);
}
```

✅ **İdeal Kullanım:** `Results` sınıfını veya uygun araçları kullanarak yanıtları yönetin.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    if (context.Request.Path == "/custom")
    {
        context.Response.StatusCode = StatusCodes.Status200OK;
        await Results.Text("Custom Response").ExecuteAsync(context);
        return;
    }

    await _next(context);
}
```

---

## 4. Exception Yönetimini İhmal Etmek

❌ **Yanlış Kullanım:** Middleware içinde yakalanmayan istisnalar bırakmak.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    // Hatalı çünkü istisna yönetimi yok
    var result = int.Parse("NotAnInt");
    await _next(context);
}
```

✅ **İdeal Kullanım:** Exception yönetimini dahil ederek istisnaları yakalayın.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    try
    {
        var result = int.Parse("NotAnInt");
    }
    catch (Exception ex)
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsync($"Hata: {ex.Message}");
        return;
    }

    await _next(context);
}
```

---

## 5. Performans Kayıplarına Yol Açan Middleware

❌ **Yanlış Kullanım:** Gereksiz loglama ve fazla işlem yapmak.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    Console.WriteLine($"Request Path: {context.Request.Path}");
    Console.WriteLine($"Headers: {string.Join(", ", context.Request.Headers.Select(h => h.Key + ": " + h.Value))}");
    await _next(context);
}
```

✅ **İdeal Kullanım:** Performans açısından kritik işlemleri minimize edin.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    if (context.Request.Path.StartsWithSegments("/debug"))
    {
        Console.WriteLine($"Request Path: {context.Request.Path}");
    }

    await _next(context);
}
```

---

## 6. Middleware'in Yeniden Kullanılamaması

❌ **Yanlış Kullanım:** Middleware'i yalnızca tek bir bağlamda çalışacak şekilde tasarlamak.

```csharp
app.UseMiddleware<MyCustomMiddleware>();
```

✅ **İdeal Kullanım:** Middleware'i esnek ve yeniden kullanılabilir hale getirin.

```csharp
public class MyCustomMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _customMessage;

    public MyCustomMiddleware(RequestDelegate next, string customMessage)
    {
        _next = next;
        _customMessage = customMessage;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine(_customMessage);
        await _next(context);
    }
}

app.UseMiddleware<MyCustomMiddleware>("Merhaba Dünya!");
```