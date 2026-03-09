# Authorization

Authorization, kimliği doğrulanmış kullanıcıların hangi kaynaklara erişebileceğini belirler; yanlış yapılandırma yetkisiz erişime ve veri sızıntısına yol açar.

---

## 1. Role-Based Authorization'ı Hardcode Etmek

❌ **Yanlış Kullanım:** Controller'da string role kontrolü yapmak.

```csharp
[HttpDelete("{id}")]
public async Task<IActionResult> DeleteOrder(int id)
{
    if (!User.IsInRole("Admin") && !User.IsInRole("SuperAdmin"))
        return Forbid();

    await _service.DeleteAsync(id);
    return NoContent();
}
```

✅ **İdeal Kullanım:** Policy-based authorization ile merkezileştirin.

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("CanDeleteOrders", policy =>
        policy.RequireRole("Admin", "SuperAdmin"))
    .AddPolicy("CanManageProducts", policy =>
        policy.RequireClaim("permission", "products.manage"));

[HttpDelete("{id}")]
[Authorize(Policy = "CanDeleteOrders")]
public async Task<IActionResult> DeleteOrder(int id)
{
    await _service.DeleteAsync(id);
    return NoContent();
}
```

---

## 2. Resource-Based Authorization Yapmamak

❌ **Yanlış Kullanım:** Kullanıcının kendi kaynağına erişimini kontrol etmemek.

```csharp
[Authorize]
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _service.GetAsync(id);
    return Ok(order); // Herhangi bir kullanıcı başkasının siparişini görebilir
}
```

✅ **İdeal Kullanım:** Resource-based authorization ile kaynak sahipliğini doğrulayın.

```csharp
public class OrderAuthorizationHandler : AuthorizationHandler<OwnerRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OwnerRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (resource.CustomerId.ToString() == userId)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

[Authorize]
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _service.GetAsync(id);
    var result = await _authorizationService.AuthorizeAsync(User, order, new OwnerRequirement());

    if (!result.Succeeded) return Forbid();
    return Ok(order);
}
```

---

## 3. IDOR (Insecure Direct Object Reference)

❌ **Yanlış Kullanım:** ID parametresini doğrulamadan kaynağa erişim vermek.

```csharp
[HttpPut("api/users/{id}/profile")]
public async Task<IActionResult> UpdateProfile(int id, UpdateProfileDto dto)
{
    await _service.UpdateProfileAsync(id, dto); // Herkes başkasının profilini güncelleyebilir
    return Ok();
}
```

✅ **İdeal Kullanım:** Kullanıcı kimliğini token'dan alarak doğrulayın.

```csharp
[HttpPut("api/users/profile")]
public async Task<IActionResult> UpdateProfile(UpdateProfileDto dto)
{
    var userId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
    await _service.UpdateProfileAsync(userId, dto);
    return Ok();
}
```

---

## 4. Claim Transformation Kullanmamak

❌ **Yanlış Kullanım:** Her endpoint'te claim'leri manuel okumak.

```csharp
[HttpGet("api/dashboard")]
public async Task<IActionResult> GetDashboard()
{
    var userId = User.FindFirst("sub")?.Value;
    var permissions = await _permissionService.GetPermissionsAsync(userId);

    if (!permissions.Contains("dashboard.view"))
        return Forbid();

    return Ok(await _dashboardService.GetAsync());
}
```

✅ **İdeal Kullanım:** Claims transformation ile claim'leri otomatik zenginleştirin.

```csharp
public class PermissionClaimsTransformation : IClaimsTransformation
{
    private readonly IPermissionService _permissionService;

    public PermissionClaimsTransformation(IPermissionService permissionService)
        => _permissionService = permissionService;

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var userId = principal.FindFirst("sub")?.Value;
        if (userId == null) return principal;

        var permissions = await _permissionService.GetPermissionsAsync(userId);
        var identity = (ClaimsIdentity)principal.Identity!;

        foreach (var permission in permissions)
        {
            identity.AddClaim(new Claim("permission", permission));
        }

        return principal;
    }
}

// Artık policy ile temiz kullanım
[Authorize(Policy = "CanViewDashboard")]
[HttpGet("api/dashboard")]
public async Task<IActionResult> GetDashboard()
{
    return Ok(await _dashboardService.GetAsync());
}
```

---

## 5. Global Authorization Filter Uygulamamak

❌ **Yanlış Kullanım:** Her controller'a `[Authorize]` eklemeyi unutmak.

```csharp
[ApiController]
public class OrdersController : ControllerBase { /* [Authorize] unutulmuş */ }

[Authorize]
[ApiController]
public class ProductsController : ControllerBase { /* Bu korunuyor */ }
```

✅ **İdeal Kullanım:** Global olarak authorize edip, public endpoint'leri explicit açın.

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());

// Tüm endpoint'ler varsayılan olarak korunur
// Sadece public endpoint'ler explicit olarak açılır
[AllowAnonymous]
[HttpPost("api/auth/login")]
public async Task<IActionResult> Login(LoginDto dto) { /* ... */ }

[AllowAnonymous]
[HttpGet("api/products")]
public async Task<IActionResult> GetProducts() { /* ... */ }
```