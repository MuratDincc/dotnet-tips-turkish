# Data Protection

Data Protection, hassas verilerin şifrelenmesi ve güvenli saklanmasını sağlar; yanlış uygulamalar veri sızıntısına ve uyumluluk ihlallerine yol açar.

---

## 1. Şifreleri Düz Metin Saklamak

❌ **Yanlış Kullanım:** Şifreleri hash'lemeden veritabanında tutmak.

```csharp
public class UserService
{
    public async Task RegisterAsync(string email, string password)
    {
        var user = new User
        {
            Email = email,
            Password = password // Düz metin şifre!
        };
        await _context.Users.AddAsync(user);
        await _context.SaveChangesAsync();
    }
}
```

✅ **İdeal Kullanım:** BCrypt veya ASP.NET Identity ile şifreleri hash'leyin.

```csharp
public class UserService
{
    private readonly IPasswordHasher<User> _passwordHasher;

    public async Task RegisterAsync(string email, string password)
    {
        var user = new User { Email = email };
        user.PasswordHash = _passwordHasher.HashPassword(user, password);

        await _context.Users.AddAsync(user);
        await _context.SaveChangesAsync();
    }

    public bool VerifyPassword(User user, string password)
    {
        var result = _passwordHasher.VerifyHashedPassword(user, user.PasswordHash, password);
        return result == PasswordVerificationResult.Success;
    }
}
```

---

## 2. Connection String'de Şifre Açık Bırakmak

❌ **Yanlış Kullanım:** Şifreleri kaynak kodunda veya config dosyasında saklamak.

```json
{
    "ConnectionStrings": {
        "Default": "Server=prod-db;Database=MyApp;User=sa;Password=P@ssw0rd123!"
    }
}
```

✅ **İdeal Kullanım:** User Secrets, environment variable veya Key Vault kullanın.

```csharp
// Development: User Secrets
// dotnet user-secrets set "ConnectionStrings:Default" "Server=...;Password=..."

// Production: Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myapp-vault.vault.azure.net/"),
    new DefaultAzureCredential());

// Veya environment variable
// ConnectionStrings__Default=Server=prod-db;Password=...
```

---

## 3. Data Protection API Kullanmamak

❌ **Yanlış Kullanım:** Hassas verileri sabit key ile şifrelemek.

```csharp
public class EncryptionService
{
    private static readonly byte[] Key = Encoding.UTF8.GetBytes("1234567890123456"); // Hardcoded key

    public string Encrypt(string data)
    {
        using var aes = Aes.Create();
        aes.Key = Key;
        // Manuel şifreleme, key yönetimi zor
    }
}
```

✅ **İdeal Kullanım:** ASP.NET Data Protection API kullanın.

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90))
    .SetApplicationName("MyApp");

public class SensitiveDataService
{
    private readonly IDataProtector _protector;

    public SensitiveDataService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("SensitiveData.v1");
    }

    public string Protect(string plainText) => _protector.Protect(plainText);
    public string Unprotect(string protectedText) => _protector.Unprotect(protectedText);
}
```

---

## 4. Log'lara Hassas Veri Yazmak

❌ **Yanlış Kullanım:** Şifre, token ve kişisel verileri log'lamak.

```csharp
public async Task LoginAsync(string email, string password)
{
    _logger.LogInformation("Login attempt: {Email}, {Password}", email, password); // Şifre log'da!

    var user = await _userService.AuthenticateAsync(email, password);
    _logger.LogInformation("Token: {Token}", user.Token); // Token log'da!
}
```

✅ **İdeal Kullanım:** Hassas verileri maskeleyerek log'layın.

```csharp
public async Task LoginAsync(string email, string password)
{
    _logger.LogInformation("Login attempt: {Email}", email);

    var user = await _userService.AuthenticateAsync(email, password);
    _logger.LogInformation("Login successful: {UserId}", user.Id);
}

// Destructuring policy ile otomatik maskeleme
public class SensitiveDataDestructuringPolicy : IDestructuringPolicy
{
    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory,
        out LogEventPropertyValue result)
    {
        if (value is LoginRequest request)
        {
            result = factory.CreatePropertyValue(new
            {
                request.Email,
                Password = "***REDACTED***"
            }, true);
            return true;
        }
        result = null;
        return false;
    }
}
```

---

## 5. HTTPS Zorunlu Kılmamak

❌ **Yanlış Kullanım:** HTTP üzerinden hassas veri iletimi.

```csharp
app.MapControllers(); // HTTP ve HTTPS ayrımı yok
// Man-in-the-middle saldırısına açık
```

✅ **İdeal Kullanım:** HTTPS'i zorunlu kılın ve HSTS uygulayın.

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();

// Program.cs
builder.Services.AddHttpsRedirection(options =>
{
    options.HttpsPort = 443;
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
});

// Strict-Transport-Security header
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});
```