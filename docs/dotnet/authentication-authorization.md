# Authentication ve Authorization

Authentication (kimlik doğrulama) ve authorization (yetkilendirme), modern web uygulamalarının temel güvenlik bileşenleridir. Yanlış uygulamalar güvenlik açıklarına, performans sorunlarına ve kullanıcı deneyimi problemlerine yol açabilir.

---

## 1. JWT'nin Yanlış Kullanımı

❌ **Yanlış Kullanım:** JWT'yi saklamak için `localStorage` kullanmak.

```javascript
localStorage.setItem("jwt", token); // Güvenlik açığına neden olabilir
```

✅ **İdeal Kullanım:** JWT'yi `HttpOnly` cookie olarak saklayarak XSS saldırılarını önleyin.

```csharp
var cookieOptions = new CookieOptions
{
    HttpOnly = true,
    Secure = true,
    SameSite = SameSiteMode.Strict
};
Response.Cookies.Append("jwt", token, cookieOptions);
```

---

## 2. Hatalı Yetkilendirme Kontrolleri

❌ **Yanlış Kullanım:** Yetkilendirme kontrollerini istemci tarafında gerçekleştirmek.

```javascript
if (user.role === "admin") {
    // Yetkili işlemler
}
```

✅ **İdeal Kullanım:** Yetkilendirme kontrollerini sunucu tarafında gerçekleştirin.

```csharp
[Authorize(Roles = "Admin")]
public IActionResult AdminEndpoint()
{
    return Ok("Yalnızca admin kullanıcılar erişebilir.");
}
```

---

## 3. Şifrelerin Yanlış Yönetimi

❌ **Yanlış Kullanım:** Şifreleri düz metin (plaintext) olarak saklamak.

```sql
INSERT INTO Users (Username, Password) VALUES ('user1', '123456');
```

✅ **İdeal Kullanım:** Şifreleri hash'leyerek güvenli bir şekilde saklayın.

```csharp
var hashedPassword = BCrypt.Net.BCrypt.HashPassword("123456");
```

---

## 4. Güvenli Olmayan Varsayılan Yapılandırmalar

❌ **Yanlış Kullanım:** HTTPS'i zorunlu kılmamak.

```csharp
app.UseAuthentication();
```

✅ **İdeal Kullanım:** HTTPS kullanımını zorunlu kılın ve güvenli yapılandırmalar uygulayın.

```csharp
app.UseHttpsRedirection();
app.UseAuthentication();
```

---

## 5. Expired Token Yönetiminin İhmal Edilmesi

❌ **Yanlış Kullanım:** Süresi dolan token'ları kontrol etmemek.

```csharp
var tokenHandler = new JwtSecurityTokenHandler();
var token = tokenHandler.ReadJwtToken(jwt);
```

✅ **İdeal Kullanım:** Token geçerliliğini doğrulayın ve süresi dolan token'ları yönetin.

```csharp
var validationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidateAudience = true,
    ValidateLifetime = true,
    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("secret_key"))
};
tokenHandler.ValidateToken(jwt, validationParameters, out SecurityToken validatedToken);
```

---

## 6. Role-Based Authorization Yanlış Kullanımı

❌ **Yanlış Kullanım:** Rolleri sabit kodlamak.

```csharp
if (user.Role == "Admin")
{
    // Yetkili işlemler
}
```

✅ **İdeal Kullanım:** Policy-based authorization uygulayın.

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});

[Authorize(Policy = "AdminOnly")]
public IActionResult AdminEndpoint()
{
    return Ok("Yalnızca admin kullanıcılar erişebilir.");
}
```

---

## 7. Open Redirect Güvenlik Açıkları

❌ **Yanlış Kullanım:** Redirect URL'lerini doğrulamadan yönlendirmek.

```csharp
return Redirect(returnUrl); // Güvenlik açığı
```

✅ **İdeal Kullanım:** Yönlendirme URL'lerini doğrulayın.

```csharp
if (Url.IsLocalUrl(returnUrl))
{
    return Redirect(returnUrl);
}
return RedirectToAction("Index", "Home");
```

---

## 8. Kullanıcı Oturumlarının Kötü Yönetimi

❌ **Yanlış Kullanım:** Kullanıcı oturumlarını manuel olarak yönetmek.

```csharp
HttpContext.Session.SetString("User", "user1");
```

✅ **İdeal Kullanım:** Identity framework gibi standart oturum yönetimi araçlarını kullanın.

```csharp
services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();
```