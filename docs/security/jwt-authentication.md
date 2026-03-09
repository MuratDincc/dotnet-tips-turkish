# JWT Authentication

JWT (JSON Web Token), stateless kimlik doğrulama sağlar; yanlış yapılandırma güvenlik açıklarına ve token sızıntılarına yol açar.

---

## 1. Zayıf Secret Key Kullanmak

❌ **Yanlış Kullanım:** Kısa ve tahmin edilebilir secret key kullanmak.

```csharp
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("mysecret")); // Çok kısa, brute-force'a açık
```

✅ **İdeal Kullanım:** Yeterli uzunlukta ve güvenli secret key kullanın.

```csharp
var secretKey = builder.Configuration["Jwt:Secret"]; // Minimum 256-bit (32 byte)
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = key,
            ClockSkew = TimeSpan.Zero
        };
    });
```

---

## 2. Token'da Hassas Veri Taşımak

❌ **Yanlış Kullanım:** JWT payload'una şifre veya kişisel veri koymak.

```csharp
var claims = new[]
{
    new Claim("email", user.Email),
    new Claim("password", user.Password),      // Şifre token'da!
    new Claim("creditCard", user.CreditCard),   // Kart bilgisi token'da!
    new Claim("address", user.Address)
};
```

✅ **İdeal Kullanım:** Token'da sadece gerekli minimum bilgiyi taşıyın.

```csharp
var claims = new[]
{
    new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
    new Claim(JwtRegisteredClaimNames.Email, user.Email),
    new Claim(ClaimTypes.Role, user.Role),
    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
};
```

---

## 3. Token Expiry Süresini Çok Uzun Tutmak

❌ **Yanlış Kullanım:** Token'ı günlerce veya haftalarca geçerli tutmak.

```csharp
var token = new JwtSecurityToken(
    expires: DateTime.UtcNow.AddDays(30), // 30 gün geçerli, çalınırsa uzun süre kötüye kullanılır
    // ...
);
```

✅ **İdeal Kullanım:** Kısa ömürlü access token ve refresh token kullanın.

```csharp
public class TokenService
{
    public TokenPair GenerateTokens(User user)
    {
        var accessToken = GenerateAccessToken(user, TimeSpan.FromMinutes(15));
        var refreshToken = GenerateRefreshToken();

        return new TokenPair(accessToken, refreshToken);
    }

    private string GenerateAccessToken(User user, TimeSpan expiry)
    {
        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: GetClaims(user),
            expires: DateTime.UtcNow.Add(expiry),
            signingCredentials: _credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    private string GenerateRefreshToken()
    {
        var randomBytes = RandomNumberGenerator.GetBytes(64);
        return Convert.ToBase64String(randomBytes);
    }
}
```

---

## 4. Token Validation Parametrelerini Gevşek Bırakmak

❌ **Yanlış Kullanım:** Validation parametrelerini devre dışı bırakmak.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = false,       // Issuer kontrol edilmiyor
    ValidateAudience = false,     // Audience kontrol edilmiyor
    ValidateLifetime = false,     // Süresi dolmuş tokenlar kabul ediliyor
    ValidateIssuerSigningKey = false // İmza doğrulanmıyor!
};
```

✅ **İdeal Kullanım:** Tüm validation parametrelerini aktif tutun.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidateAudience = true,
    ValidateLifetime = true,
    ValidateIssuerSigningKey = true,
    ValidIssuer = builder.Configuration["Jwt:Issuer"],
    ValidAudience = builder.Configuration["Jwt:Audience"],
    IssuerSigningKey = new SymmetricSecurityKey(
        Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"])),
    ClockSkew = TimeSpan.Zero // Varsayılan 5 dakika toleransı kaldır
};
```

---

## 5. Token Revocation Mekanizması Olmamak

❌ **Yanlış Kullanım:** Çıkış yapıldığında token'ı geçersiz kılamamak.

```csharp
[HttpPost("logout")]
public IActionResult Logout()
{
    return Ok("Çıkış yapıldı");
    // Token hala geçerli! Süresi dolana kadar kullanılabilir
}
```

✅ **İdeal Kullanım:** Token blacklist veya refresh token revocation uygulayın.

```csharp
[HttpPost("logout")]
public async Task<IActionResult> Logout()
{
    var token = HttpContext.Request.Headers["Authorization"]
        .ToString().Replace("Bearer ", "");

    var jti = User.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
    await _cache.SetStringAsync($"blacklist:{jti}", "revoked",
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15)
        });

    return Ok("Çıkış yapıldı");
}

// Middleware ile blacklist kontrolü
public class TokenBlacklistMiddleware
{
    public async Task InvokeAsync(HttpContext context, IDistributedCache cache)
    {
        var jti = context.User.FindFirst(JwtRegisteredClaimNames.Jti)?.Value;
        if (jti != null && await cache.GetStringAsync($"blacklist:{jti}") != null)
        {
            context.Response.StatusCode = 401;
            return;
        }
        await _next(context);
    }
}
```