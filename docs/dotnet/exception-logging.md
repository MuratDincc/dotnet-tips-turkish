# Exception Logging

Exception logging, bir uygulamada meydana gelen hataları anlamak ve düzeltmek için hayati önem taşır. Ancak, kötü yapılandırılmış veya yetersiz logging stratejileri, hata ayıklamayı zorlaştırabilir.

---

## 1. Hataların Günlüğe Kaydedilmemesi

❌ **Yanlış Kullanım:** Hataları yalnızca konsola yazdırmak.

```csharp
try
{
    // İşlem
}
catch (Exception ex)
{
    Console.WriteLine($"Hata: {ex.Message}");
}
```

✅ **İdeal Kullanım:** Günlükleme framework'leri ile hataları detaylı şekilde kaydedin.

```csharp
var logger = LoggerFactory.Create(builder => builder.AddConsole()).CreateLogger("AppLogger");

try
{
    // İşlem
}
catch (Exception ex)
{
    logger.LogError(ex, "Bir hata oluştu.");
}
```

---

## 2. Yetersiz Hata Mesajları

❌ **Yanlış Kullanım:** Hataların bağlamını (context) belirtmemek.

```csharp
catch (Exception ex)
{
    logger.LogError(ex.Message); // Bağlam eksik
}
```

✅ **İdeal Kullanım:** Hata bağlamını açıkça belirterek daha fazla bilgi sağlayın.

```csharp
catch (Exception ex)
{
    logger.LogError(ex, "Veritabanı işlemi sırasında bir hata oluştu.");
}
```

---

## 3. Global Exception Logging'i Atlamak

❌ **Yanlış Kullanım:** Global hata yönetimini ve günlüklemeyi ihmal etmek.

```csharp
app.MapGet("/", () => throw new Exception("Hata!")); // Global yönetim yok
```

✅ **İdeal Kullanım:** Global exception handler ile tüm hataları yakalayın.

```csharp
app.UseExceptionHandler("/error");

app.Map("/error", (HttpContext context) =>
{
    var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
    logger.LogError(exception, "Global bir hata yakalandı.");
    return Results.Problem(detail: exception?.Message, statusCode: 500);
});
```

---

## 4. Hataların Tekrar Edilerek Kaydedilmesi

❌ **Yanlış Kullanım:** Aynı hatayı birden fazla kez kaydetmek.

```csharp
catch (Exception ex)
{
    logger.LogError(ex, "Hata!");
    throw; // Yeniden kaydedilir.
}
```

✅ **İdeal Kullanım:** Hataları yalnızca bir kez kaydedin.

```csharp
catch (Exception ex) when (LogException(ex))
{
    throw;
}

static bool LogException(Exception ex)
{
    logger.LogError(ex, "Hata kaydedildi.");
    return false;
}
```

---

## 5. Hassas Bilgilerin Günlüğe Kaydedilmesi

❌ **Yanlış Kullanım:** Kullanıcı verilerini veya hassas bilgileri loglamak.

```csharp
catch (Exception ex)
{
    logger.LogError($"Hata: {ex.Message}, Kullanıcı: {user.Password}"); // Güvenlik açığı
}
```

✅ **İdeal Kullanım:** Hassas bilgileri hariç tutun.

```csharp
catch (Exception ex)
{
    logger.LogError(ex, "Hata oluştu.");
}
```

---

## 6. Logging Seviyelerini Yanlış Kullanmak

❌ **Yanlış Kullanım:** Tüm hataları aynı seviyede kaydetmek.

```csharp
logger.LogError("Hata!"); // Her şey Error olarak loglanmış.
```

✅ **İdeal Kullanım:** Doğru log seviyelerini kullanın.

```csharp
logger.LogInformation("Bilgilendirme: İşlem başladı.");
logger.LogWarning("Uyarı: Beklenmeyen bir durum.");
logger.LogError("Hata: Bir istisna yakalandı.");
```

---

## 7. Log Management Sistemlerini Kullanmamak

❌ **Yanlış Kullanım:** Lokal loglama ile sınırlı kalmak.

```csharp
logger.LogError("Hata kaydedildi.");
```

✅ **İdeal Kullanım:** Merkezi log yönetimi araçlarını kullanın.

- **Azure Application Insights**
- **Elastic Stack (ELK)**
- **Sentry**
- **Loggly**
- **Seq**
- **Splunk**
- **Datadog**
- **Raygun**
- **New Relic**
- **Serilog**
- **NLog**
- **Log4Net**
- **Graylog**