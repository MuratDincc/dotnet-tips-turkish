# OpenID Connect ve Token Yönetimi

OpenID Connect, kimlik doğrulama için standart bir protokoldür; yanlış token yönetimi güvenlik açıklarına ve kötü kullanıcı deneyimine yol açar.

---

## 1. Token'ı LocalStorage'da Saklamak

❌ **Yanlış Kullanım:** JWT token'ı tarayıcı localStorage'ında tutmak.

```javascript
// Frontend - XSS saldırısına açık
localStorage.setItem("access_token", response.token);
fetch("/api/orders", {
    headers: { "Authorization": `Bearer ${localStorage.getItem("access_token")}` }
});
```

✅ **İdeal Kullanım:** HttpOnly cookie ile token yönetimi yapın.

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie(options =>
{
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.ExpireTimeSpan = TimeSpan.FromHours(1);
    options.SlidingExpiration = true;
})
.AddOpenIdConnect(options =>
{
    options.Authority = "https://auth.example.com";
    options.ClientId = "web-app";
    options.ClientSecret = builder.Configuration["Auth:ClientSecret"];
    options.ResponseType = "code";
    options.SaveTokens = true;
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("api");
});
```

---

## 2. Refresh Token Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Access token süresi dolduğunda kullanıcıyı tekrar login'e yönlendirmek.

```csharp
app.MapGet("/api/data", async (HttpContext context) =>
{
    var token = await context.GetTokenAsync("access_token");
    // Token expire olduysa 401 döner, kullanıcı tekrar giriş yapmak zorunda
    _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
    return await _client.GetFromJsonAsync<Data>("/data");
});
```

✅ **İdeal Kullanım:** Otomatik token yenileme mekanizması kurun.

```csharp
public class TokenRefreshHandler : DelegatingHandler
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ITokenService _tokenService;

    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var context = _httpContextAccessor.HttpContext!;
        var accessToken = await context.GetTokenAsync("access_token");
        var expiresAt = await context.GetTokenAsync("expires_at");

        if (DateTimeOffset.Parse(expiresAt!) < DateTimeOffset.UtcNow.AddMinutes(1))
        {
            var refreshToken = await context.GetTokenAsync("refresh_token");
            var newTokens = await _tokenService.RefreshAsync(refreshToken!);

            var authInfo = await context.AuthenticateAsync();
            authInfo.Properties!.UpdateTokenValue("access_token", newTokens.AccessToken);
            authInfo.Properties!.UpdateTokenValue("refresh_token", newTokens.RefreshToken);
            authInfo.Properties!.UpdateTokenValue("expires_at",
                newTokens.ExpiresAt.ToString("o"));

            await context.SignInAsync(authInfo.Principal!, authInfo.Properties);
            accessToken = newTokens.AccessToken;
        }

        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
        return await base.SendAsync(request, ct);
    }
}
```

---

## 3. Claims Mapping Yapmamak

❌ **Yanlış Kullanım:** Claim'leri doğrudan token'dan okumaya çalışmak.

```csharp
app.MapGet("/api/profile", (ClaimsPrincipal user) =>
{
    var name = user.FindFirst("name")?.Value;           // null olabilir
    var email = user.FindFirst("email")?.Value;          // Farklı provider'larda farklı claim adı
    var role = user.FindFirst("role")?.Value;             // Mapping olmadan boş
    return new { name, email, role };
});
```

✅ **İdeal Kullanım:** Claims transformation ile standart claim mapping yapın.

```csharp
public class CustomClaimsTransformation : IClaimsTransformation
{
    private readonly IUserService _userService;

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = (ClaimsIdentity)principal.Identity!;
        var externalId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (externalId is null) return principal;

        var user = await _userService.GetByExternalIdAsync(externalId);
        if (user is null) return principal;

        identity.AddClaim(new Claim("app_user_id", user.Id.ToString()));
        identity.AddClaim(new Claim(ClaimTypes.Role, user.Role));

        foreach (var permission in user.Permissions)
            identity.AddClaim(new Claim("permission", permission));

        return principal;
    }
}

builder.Services.AddTransient<IClaimsTransformation, CustomClaimsTransformation>();
```

---

## 4. Tek Bir Authentication Scheme Kullanmak

❌ **Yanlış Kullanım:** Sadece bir kimlik doğrulama yöntemi desteklemek.

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer(options =>
    {
        options.Authority = "https://auth.example.com";
    });
// Sadece JWT, API key veya farklı provider desteği yok
```

✅ **İdeal Kullanım:** Birden fazla scheme ile esnek kimlik doğrulama yapın.

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://auth.example.com";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = true,
            ValidAudience = "api"
        };
    })
    .AddScheme<ApiKeyAuthOptions, ApiKeyAuthHandler>("ApiKey", null);

// Policy-based scheme seçimi
builder.Services.AddAuthorization(options =>
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .AddAuthenticationSchemes("Bearer", "ApiKey")
        .Build();

    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin")
              .AddAuthenticationSchemes("Bearer"));
});
```

---

## 5. Token Validation'ı Eksik Yapmak

❌ **Yanlış Kullanım:** Token doğrulama parametrelerini atlamak.

```csharp
builder.Services.AddAuthentication().AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = false,      // Herhangi bir issuer kabul edilir
        ValidateAudience = false,    // Herhangi bir audience kabul edilir
        ValidateLifetime = false,    // Süresi dolmuş token kabul edilir
        ValidateIssuerSigningKey = false
    };
});
```

✅ **İdeal Kullanım:** Tüm token doğrulama parametrelerini aktif edin.

```csharp
builder.Services.AddAuthentication().AddJwtBearer(options =>
{
    options.Authority = "https://auth.example.com";
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = "https://auth.example.com",
        ValidateAudience = true,
        ValidAudience = "api",
        ValidateLifetime = true,
        ClockSkew = TimeSpan.FromMinutes(1),
        ValidateIssuerSigningKey = true,
        RequireExpirationTime = true,
        RequireSignedTokens = true
    };

    options.Events = new JwtBearerEvents
    {
        OnTokenValidated = async context =>
        {
            var userService = context.HttpContext.RequestServices
                .GetRequiredService<IUserService>();
            var userId = context.Principal!.FindFirst(ClaimTypes.NameIdentifier)?.Value;

            if (userId is null || !await userService.IsActiveAsync(userId))
                context.Fail("Kullanıcı aktif değil");
        }
    };
});
```
