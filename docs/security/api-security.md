# API Security

API güvenliği, uygulamanın dış tehditlere karşı korunmasını sağlar; eksik güvenlik önlemleri veri sızıntısına ve servis kesintisine yol açar.

---

## 1. CORS'u Herkese Açmak

❌ **Yanlış Kullanım:** Tüm origin'lere izin vermek.

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()     // Herkes erişebilir
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

✅ **İdeal Kullanım:** Sadece bilinen origin'lere izin verin.

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(
                "https://myapp.com",
                "https://admin.myapp.com")
              .AllowCredentials()
              .WithHeaders("Authorization", "Content-Type")
              .WithMethods("GET", "POST", "PUT", "DELETE");
    });
});
```

---

## 2. Rate Limiting Yapmamak

❌ **Yanlış Kullanım:** API endpoint'lerini sınırsız istek almaya açık bırakmak.

```csharp
app.MapControllers(); // Rate limit yok, brute-force ve DDoS'a açık
```

✅ **İdeal Kullanım:** Rate limiting ile endpoint'leri koruyun.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("general", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
    });

    options.AddFixedWindowLimiter("auth", limiter =>
    {
        limiter.PermitLimit = 5;
        limiter.Window = TimeSpan.FromMinutes(1);
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

app.MapPost("/api/auth/login", Login).RequireRateLimiting("auth");
app.MapControllers().RequireRateLimiting("general");
```

---

## 3. Input Validation Yapmamak

❌ **Yanlış Kullanım:** Kullanıcı girdisini doğrulamadan işlemek.

```csharp
[HttpGet("api/users")]
public async Task<IActionResult> Search([FromQuery] string query)
{
    var users = await _context.Users
        .FromSqlRaw($"SELECT * FROM Users WHERE Name LIKE '%{query}%'") // SQL Injection!
        .ToListAsync();
    return Ok(users);
}
```

✅ **İdeal Kullanım:** Parametreli sorgular ve input validation kullanın.

```csharp
[HttpGet("api/users")]
public async Task<IActionResult> Search([FromQuery] string query)
{
    if (string.IsNullOrWhiteSpace(query) || query.Length > 100)
        return BadRequest("Geçersiz arama terimi.");

    var users = await _context.Users
        .Where(u => EF.Functions.Like(u.Name, $"%{query}%"))
        .Take(50)
        .ToListAsync();

    return Ok(users);
}
```

---

## 4. Security Headers Eklememek

❌ **Yanlış Kullanım:** Varsayılan HTTP header'ları ile çalışmak.

```csharp
app.MapControllers();
// X-Content-Type-Options, X-Frame-Options, CSP header'ları eksik
```

✅ **İdeal Kullanım:** Güvenlik header'larını middleware ile ekleyin.

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Append("Content-Security-Policy", "default-src 'self'");
    context.Response.Headers.Remove("Server");
    context.Response.Headers.Remove("X-Powered-By");

    await next();
});

app.UseHsts();
app.UseHttpsRedirection();
```

---

## 5. Sensitive Data Exposure

❌ **Yanlış Kullanım:** Hata detaylarını ve iç yapıyı istemciye açmak.

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    try
    {
        var user = await _context.Users
            .Include(u => u.Password)       // Şifre hash'i döner
            .Include(u => u.SecurityStamp)  // İç güvenlik bilgisi
            .FirstOrDefaultAsync(u => u.Id == id);
        return Ok(user);
    }
    catch (Exception ex)
    {
        return StatusCode(500, new { error = ex.ToString() }); // Stack trace sızar
    }
}
```

✅ **İdeal Kullanım:** DTO ile sadece gerekli veriyi dönün, hataları maskelleyin.

```csharp
public record UserDto(int Id, string Name, string Email);

[HttpGet("{id}")]
public async Task<IActionResult> GetUser(int id)
{
    var user = await _service.GetByIdAsync(id);
    if (user == null) return NotFound();

    return Ok(new UserDto(user.Id, user.Name, user.Email));
}

// Global exception handler hata detaylarını gizler
public class GlobalExceptionMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Beklenmeyen hata");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new
            {
                message = "Bir hata oluştu. Lütfen daha sonra tekrar deneyin."
            });
        }
    }
}
```