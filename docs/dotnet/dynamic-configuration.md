# Dynamic Configuration: Kötü ve İdeal Kullanım

Dynamic configuration, uygulamanızın yapılandırma ayarlarını çalışma zamanında değiştirme yeteneği sağlar. Yanlış kullanılan dinamik yapılandırmalar, veri tutarsızlıklarına ve beklenmedik davranışlara yol açabilir.

---

## 1. Sabit Kodlanmış Yapılandırmalar

❌ **Yanlış Kullanım:** Yapılandırmaları sabit kodlamak.

```csharp
public class AppConfig
{
    public const string ConnectionString = "Server=localhost;Database=MyApp;User=admin;Password=1234;";
}
```

✅ **İdeal Kullanım:** Yapılandırmaları bir dosya veya çevresel değişkenlerde tutarak dinamik hale getirin.

**appsettings.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;User=admin;Password=1234;"
  }
}
```

**Kullanım:**
```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
```

---

## 2. Hassas Bilgileri Güvenli Bir Şekilde Yönetmemek

❌ **Yanlış Kullanım:** Hassas bilgileri yapılandırma dosyalarında düz metin olarak saklamak.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;User=admin;Password=1234;"
  }
}
```

✅ **İdeal Kullanım:** Hassas bilgileri güvenli bir şekilde yönetmek için bir secret management aracı kullanın.

**HashiCorp Vault Kullanımı:**

```csharp
builder.Configuration.AddVault(options =>
{
    options.Address = "https://vault.example.com";
    options.Token = Environment.GetEnvironmentVariable("VAULT_TOKEN");
});
```

**Azure Key Vault Kullanımı:**

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://mykeyvault.vault.azure.net/"),
    new DefaultAzureCredential()
);
```

---

## 3. Dinamik Yapılandırmayı İzlememek

❌ **Yanlış Kullanım:** Yapılandırma değişikliklerini izlememek.

✅ **İdeal Kullanım:** Dinamik yapılandırma değişikliklerini izlemek için yapılandırma sağlayıcılarını kullanın.

**Azure App Configuration Örneği:**
```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect("ConnectionString")
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("AppSettings:Sentinel", refreshAll: true);
           });
});
```

---

## 4. Performans Üzerindeki Etkileri Göz Ardı Etmek

❌ **Yanlış Kullanım:** Yapılandırmaların sürekli olarak okunması.

```csharp
var setting = builder.Configuration["AppSettings:SettingKey"]; // Sürekli çağrılar
```

✅ **İdeal Kullanım:** Yapılandırmaları bir cache mekanizması ile optimize edin.

```csharp
public class MyService
{
    private readonly IConfiguration _configuration;
    private string _cachedSetting;

    public MyService(IConfiguration configuration)
    {
        _configuration = configuration;
        _cachedSetting = _configuration["AppSettings:SettingKey"];
    }
}
```