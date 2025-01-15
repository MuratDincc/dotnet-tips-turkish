# Configuration Management (Yapılandırma Yönetimi)

Uygulama yapılandırması, bir yazılımın farklı ortamlarda (geliştirme, test, üretim) doğru şekilde çalışmasını sağlamak için kritik öneme sahiptir. Yanlış yapılandırma yönetimi, güvenlik açıklarına veya uygulama hatalarına yol açabilir.

---

## 1. Sabit Kodlanmış (Hardcoded) Değerler Kullanımı

❌ **Yanlış Kullanım:** Yapılandırmaları doğrudan kod içinde tanımlamak.

```csharp
public class DatabaseConfig
{
    public string ConnectionString => "Server=localhost;Database=MyApp;User=admin;Password=password;";
}
```

✅ **İdeal Kullanım:** `appsettings.json` veya çevresel değişkenler kullanarak yapılandırmaları dışsallaştırın.

**appsettings.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;User=admin;Password=password;"
  }
}
```

**Kullanım:**
```csharp
var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
```

---

## 2. Çevresel Yapılandırmaların Yetersiz Yönetimi

❌ **Yanlış Kullanım:** Tüm ortamlar için aynı yapılandırmayı kullanmak.

```csharp
{
  "Environment": "Production",
  "Logging": {
    "LogLevel": "Information"
  }
}
```

✅ **İdeal Kullanım:** Ortam bazlı yapılandırmaları ayrı dosyalarla yönetin.

**appsettings.Development.json:**
```json
{
  "Environment": "Development",
  "Logging": {
    "LogLevel": "Debug"
  }
}
```

**appsettings.Production.json:**
```json
{
  "Environment": "Production",
  "Logging": {
    "LogLevel": "Error"
  }
}
```

---

## 3. Gizli Bilgilerin Açığa Çıkarılması

❌ **Yanlış Kullanım:** API anahtarları veya şifreleri açık şekilde saklamak.

```csharp
{
  "APIKey": "12345-secret-key"
}
```

✅ **İdeal Kullanım:** Gizli bilgileri çevresel değişkenlerde veya güvenli bir yapılandırma hizmetinde saklayın.

**Kullanım:**
```csharp
var apiKey = Environment.GetEnvironmentVariable("MY_APP_API_KEY");
```

---

## 4. Yapılandırma Değişikliklerini Yeniden Dağıtım Gerektirmek

❌ **Yanlış Kullanım:** Yapılandırma değişiklikleri için uygulamanın yeniden başlatılması.

```csharp
public class Config
{
    public string SomeSetting { get; set; } = "DefaultValue";
}
```

✅ **İdeal Kullanım:** Dinamik yapılandırma yüklemeleri yapın.

**Örnek:** Azure App Configuration veya diğer dinamik yapılandırma araçlarını kullanın.

```csharp
builder.Configuration.AddAzureAppConfiguration("ConnectionString");
```

---

## 5. Gereksiz Karmaşıklık Yaratmak

❌ **Yanlış Kullanım:** Gereksiz yapılandırma anahtarları ve karmaşıklık.

```json
{
  "AppSettings": {
    "Feature1": {
      "Enabled": true,
      "MaxItems": 10,
      "Timeout": 5000
    },
    "Feature2": {
      "Enabled": false,
      "MaxItems": 5,
      "Timeout": 2000
    }
  }
}
```

✅ **İdeal Kullanım:** Sade ve okunabilir yapılandırmalar oluşturun.

```json
{
  "Features": [
    {
      "Name": "Feature1",
      "Enabled": true,
      "MaxItems": 10,
      "Timeout": 5000
    },
    {
      "Name": "Feature2",
      "Enabled": false,
      "MaxItems": 5,
      "Timeout": 2000
    }
  ]
}
```

---

## 6. Ortam Değişkenlerinin Yanlış Kullanımı

❌ **Yanlış Kullanım:** Ortam değişkenlerini doğrudan ve düzensiz bir şekilde okumak.

```csharp
var setting = Environment.GetEnvironmentVariable("MY_APP_SETTING");
```

✅ **İdeal Kullanım:** Ortam değişkenlerini yapılandırma ile birleştirin.

```csharp
builder.Configuration.AddEnvironmentVariables();
var setting = builder.Configuration["MY_APP_SETTING"];
```