# Feature Flags ve Gradual Rollout

Feature flag'ler, özellikleri kod deploy'undan bağımsız olarak açıp kapatmayı sağlar; yanlış kullanım bayrak karmaşasına ve test edilemeyen koda yol açar.

---

## 1. Hardcoded Feature Toggle Kullanmak

❌ **Yanlış Kullanım:** Özellikleri if/else ile kontrol etmek.

```csharp
app.MapGet("/api/products", async (IProductService service) =>
{
    var useNewAlgorithm = true; // Değiştirmek için yeniden deploy gerekir
    if (useNewAlgorithm)
        return await service.GetWithNewAlgorithmAsync();
    else
        return await service.GetAllAsync();
});
```

✅ **İdeal Kullanım:** Microsoft.FeatureManagement ile feature flag kullanın.

```csharp
builder.Services.AddFeatureManagement();

// appsettings.json
// {
//   "FeatureManagement": {
//     "NewSearchAlgorithm": true,
//     "BetaDashboard": false
//   }
// }

app.MapGet("/api/products", async (
    IFeatureManager featureManager,
    IProductService service) =>
{
    if (await featureManager.IsEnabledAsync("NewSearchAlgorithm"))
        return TypedResults.Ok(await service.GetWithNewAlgorithmAsync());

    return TypedResults.Ok(await service.GetAllAsync());
});
```

---

## 2. Feature Gate Kullanmamak

❌ **Yanlış Kullanım:** Her endpoint'te manuel feature check yapmak.

```csharp
app.MapGet("/api/beta/dashboard", async (IFeatureManager fm, IDashboardService service) =>
{
    if (!await fm.IsEnabledAsync("BetaDashboard"))
        return Results.NotFound();
    return Results.Ok(await service.GetBetaDashboardAsync());
});

app.MapGet("/api/beta/reports", async (IFeatureManager fm, IReportService service) =>
{
    if (!await fm.IsEnabledAsync("BetaDashboard"))
        return Results.NotFound();
    return Results.Ok(await service.GetBetaReportsAsync());
});
```

✅ **İdeal Kullanım:** FeatureGate attribute ile endpoint'leri koruyun.

```csharp
// Controller ile
[FeatureGate("BetaDashboard")]
public class BetaDashboardController : ControllerBase
{
    [HttpGet("/api/beta/dashboard")]
    public async Task<IActionResult> GetDashboard() => Ok(await _service.GetAsync());

    [HttpGet("/api/beta/reports")]
    public async Task<IActionResult> GetReports() => Ok(await _service.GetReportsAsync());
}

// Minimal API ile
var beta = app.MapGroup("/api/beta")
    .AddEndpointFilter(async (context, next) =>
    {
        var fm = context.HttpContext.RequestServices.GetRequiredService<IFeatureManager>();
        if (!await fm.IsEnabledAsync("BetaDashboard"))
            return TypedResults.NotFound();
        return await next(context);
    });

beta.MapGet("/dashboard", async (IDashboardService service) =>
    TypedResults.Ok(await service.GetAsync()));
```

---

## 3. Percentage Rollout Yapmamak

❌ **Yanlış Kullanım:** Özelliği herkese aynı anda açmak.

```csharp
// appsettings.json - Ya herkese açık ya herkese kapalı
// { "FeatureManagement": { "NewCheckout": true } }
```

✅ **İdeal Kullanım:** Kademeli rollout ile riski azaltın.

```csharp
// appsettings.json
// {
//   "FeatureManagement": {
//     "NewCheckout": {
//       "EnabledFor": [
//         {
//           "Name": "Percentage",
//           "Parameters": { "Value": 25 }
//         }
//       ]
//     }
//   }
// }

// Kullanıcı bazlı targeting
// {
//   "FeatureManagement": {
//     "PremiumFeature": {
//       "EnabledFor": [
//         {
//           "Name": "Targeting",
//           "Parameters": {
//             "Audience": {
//               "Users": ["user1@example.com", "user2@example.com"],
//               "Groups": [
//                 { "Name": "Beta", "RolloutPercentage": 50 }
//               ],
//               "DefaultRolloutPercentage": 10
//             }
//           }
//         }
//       ]
//     }
//   }
// }

// Targeting context sağlama
builder.Services.AddFeatureManagement()
    .WithTargeting<HttpContextTargetingContextAccessor>();

public class HttpContextTargetingContextAccessor : ITargetingContextAccessor
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public ValueTask<TargetingContext> GetContextAsync()
    {
        var user = _httpContextAccessor.HttpContext?.User;
        return new ValueTask<TargetingContext>(new TargetingContext
        {
            UserId = user?.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            Groups = user?.FindAll("group").Select(c => c.Value) ?? Array.Empty<string>()
        });
    }
}
```

---

## 4. Feature Flag'leri Temizlememek

❌ **Yanlış Kullanım:** Eski flag'lerin kodda kalması.

```csharp
if (await featureManager.IsEnabledAsync("NewSearch_v1")) { /* ... */ }
if (await featureManager.IsEnabledAsync("NewSearch_v2")) { /* ... */ }
if (await featureManager.IsEnabledAsync("NewSearch_v3_final")) { /* ... */ }
if (await featureManager.IsEnabledAsync("NewSearch_v3_final_REAL")) { /* ... */ }
// Hangi flag aktif? Kod okunamaz hale gelir
```

✅ **İdeal Kullanım:** Feature flag yaşam döngüsünü yönetin.

```csharp
// Feature flag'leri merkezi enum ile tanımlayın
public static class FeatureFlags
{
    public const string NewCheckout = "NewCheckout";
    public const string BetaDashboard = "BetaDashboard";

    // Deprecated: 2024-Q2'de kaldırılacak
    // public const string OldPayment = "OldPayment";
}

// Kullanım
if (await featureManager.IsEnabledAsync(FeatureFlags.NewCheckout))
{
    // yeni checkout
}

// Flag tamamen açıldıktan sonra:
// 1. Flag kontrolünü kaldırın, sadece yeni kodu bırakın
// 2. Eski kodu silin
// 3. Flag'i konfigürasyondan kaldırın
```

---

## 5. Feature Flag'leri Test Etmemek

❌ **Yanlış Kullanım:** Feature flag durumlarını test etmemek.

```csharp
[Fact]
public async Task GetProducts_ReturnsProducts()
{
    var result = await _service.GetAllAsync();
    Assert.NotEmpty(result);
    // Feature flag açık/kapalı senaryoları test edilmiyor
}
```

✅ **İdeal Kullanım:** Her flag durumu için test yazın.

```csharp
public class ProductServiceTests
{
    [Theory]
    [InlineData(true)]
    [InlineData(false)]
    public async Task GetProducts_RespectsFeatureFlag(bool flagEnabled)
    {
        var featureManager = new Mock<IFeatureManager>();
        featureManager
            .Setup(f => f.IsEnabledAsync("NewSearchAlgorithm"))
            .ReturnsAsync(flagEnabled);

        var service = new ProductService(_context, featureManager.Object);
        var result = await service.GetAllAsync();

        Assert.NotEmpty(result);
        if (flagEnabled)
            featureManager.Verify(f => f.IsEnabledAsync("NewSearchAlgorithm"), Times.Once);
    }
}

// Integration test ile
public class FeatureFlagIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task BetaEndpoint_Returns404_WhenFlagDisabled()
    {
        var client = _factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.AddSingleton<IFeatureManager>(new TestFeatureManager(
                    new Dictionary<string, bool> { ["BetaDashboard"] = false }));
            });
        }).CreateClient();

        var response = await client.GetAsync("/api/beta/dashboard");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```
