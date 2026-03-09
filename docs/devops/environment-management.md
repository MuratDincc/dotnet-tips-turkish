# Environment Management

Ortam yönetimi, uygulamanın farklı ortamlarda tutarlı çalışmasını sağlar; yanlış yapılandırma güvenlik açıklarına ve ortam çakışmalarına yol açar.

---

## 1. Hardcoded Connection String

❌ **Yanlış Kullanım:** Bağlantı bilgilerini kod içinde sabitlemek.

```csharp
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer("Server=prod-db;Database=MyApp;User=sa;Password=Secret123!");
    }
}
```

✅ **İdeal Kullanım:** Ortam bazlı konfigürasyon ile yönetin.

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// appsettings.Development.json
{
    "ConnectionStrings": {
        "Default": "Server=localhost;Database=MyApp_Dev;Trusted_Connection=true"
    }
}

// Production'da environment variable ile override
// ConnectionStrings__Default=Server=prod-db;Database=MyApp;...
```

---

## 2. Secrets'ı appsettings'te Tutmak

❌ **Yanlış Kullanım:** Gizli bilgileri kaynak kodunda saklamak.

```json
{
    "Jwt": {
        "Secret": "MySuperSecretJwtKey2024!",
        "Issuer": "MyApp"
    },
    "Stripe": {
        "ApiKey": "sk_live_1234567890"
    }
}
```

✅ **İdeal Kullanım:** User Secrets (development) ve vault (production) kullanın.

```csharp
// Development
// dotnet user-secrets set "Jwt:Secret" "MySuperSecretJwtKey2024!"

// Production - Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
    new DefaultAzureCredential());

// Veya environment variable
// Jwt__Secret=MySuperSecretJwtKey2024!
```

---

## 3. Ortam Kontrolü Yapmamak

❌ **Yanlış Kullanım:** Ortam farkını kontrol etmeden kod çalıştırmak.

```csharp
app.UseDeveloperExceptionPage(); // Production'da detaylı hata gösterir - güvenlik açığı
app.UseSwagger();                // Production'da API dökümantasyonu açık kalır
```

✅ **İdeal Kullanım:** Ortama göre middleware'leri koşullu yapılandırın.

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

if (!app.Environment.IsProduction())
{
    app.MapGet("/debug/config", (IConfiguration config) =>
        config.GetDebugView());
}
```

---

## 4. Loglama Seviyesini Ortama Göre Ayarlamamak

❌ **Yanlış Kullanım:** Tüm ortamlarda aynı log seviyesi kullanmak.

```json
{
    "Logging": {
        "LogLevel": {
            "Default": "Debug"
        }
    }
}
// Production'da debug logları performans sorunu yaratır
```

✅ **İdeal Kullanım:** Ortam bazlı log seviyesi belirleyin.

```json
// appsettings.Development.json
{
    "Logging": {
        "LogLevel": {
            "Default": "Debug",
            "Microsoft.EntityFrameworkCore": "Information"
        }
    }
}

// appsettings.Production.json
{
    "Logging": {
        "LogLevel": {
            "Default": "Warning",
            "Microsoft": "Warning",
            "MyApp": "Information"
        }
    }
}
```

---

## 5. Feature Toggle ile Ortam Yönetimi

❌ **Yanlış Kullanım:** Özellikleri if-else ile ortama göre açıp kapatmak.

```csharp
if (Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") == "Production")
{
    // Yeni ödeme sistemi kapalı
}
else
{
    // Yeni ödeme sistemi açık
}
```

✅ **İdeal Kullanım:** Feature flag sistemi ile yönetin.

```csharp
builder.Services.AddFeatureManagement(builder.Configuration.GetSection("FeatureFlags"));

// appsettings.json
{
    "FeatureFlags": {
        "NewPaymentSystem": false,
        "V2Api": true
    }
}

[FeatureGate("NewPaymentSystem")]
[HttpPost("api/payments/v2")]
public async Task<IActionResult> ProcessPaymentV2(PaymentRequest request)
{
    // Yeni ödeme sistemi
}

// Veya servis içinde
public class PaymentService
{
    private readonly IFeatureManager _featureManager;

    public async Task ProcessAsync(Payment payment)
    {
        if (await _featureManager.IsEnabledAsync("NewPaymentSystem"))
            await ProcessWithNewSystemAsync(payment);
        else
            await ProcessWithLegacyAsync(payment);
    }
}
```