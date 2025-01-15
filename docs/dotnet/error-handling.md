# Error Handling

Hata yönetimi (error handling), yazılımın güvenilirliğini ve kararlılığını artıran önemli bir unsurdur. Ancak, hataların doğru yönetilmemesi, hata ayıklamayı zorlaştırabilir ve uygulama performansını olumsuz etkileyebilir.

---

## 1. Hataların Genel Olarak Yakalanması

❌ **Yanlış Kullanım:** Tüm hataları `Exception` sınıfıyla yakalamak.

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

✅ **İdeal Kullanım:** Spesifik hata türlerini yakalayarak daha detaylı hata yönetimi gerçekleştirin.

```csharp
try
{
    // İşlem
}
catch (NullReferenceException ex)
{
    Console.WriteLine($"Null reference hatası: {ex.Message}");
}
catch (FileNotFoundException ex)
{
    Console.WriteLine($"Dosya bulunamadı: {ex.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Beklenmeyen bir hata: {ex.Message}");
}
```

---

## 2. Finally Bloğunun Yanlış Kullanımı

❌ **Yanlış Kullanım:** `finally` bloğu içinde kodun kontrol dışı bırakılması.

```csharp
try
{
    // İşlem
}
finally
{
    throw new InvalidOperationException("Hatalı işlem");
}
```

✅ **İdeal Kullanım:** Kaynakların doğru şekilde serbest bırakılmasını sağlayın.

```csharp
FileStream? file = null;

try
{
    file = new FileStream("data.txt", FileMode.Open);
    // İşlem
}
finally
{
    file?.Dispose();
}
```

---

## 3. Asenkron Hata Yönetimi

❌ **Yanlış Kullanım:** Asenkron hataları `await` etmeden yakalamaya çalışmak.

```csharp
try
{
    DoAsync(); // await eksik
}
catch (Exception ex)
{
    Console.WriteLine($"Hata: {ex.Message}");
}
```

✅ **İdeal Kullanım:** Asenkron işlemleri `await` ile yakalayarak doğru hata yönetimi sağlayın.

```csharp
try
{
    await DoAsync();
}
catch (Exception ex)
{
    Console.WriteLine($"Asenkron hata: {ex.Message}");
}
```

---

## 4. Hataların Günlüğe Kaydedilmemesi

❌ **Yanlış Kullanım:** Hataları yalnızca konsola yazdırmak.

```csharp
catch (Exception ex)
{
    Console.WriteLine($"Hata: {ex.Message}");
}
```

✅ **İdeal Kullanım:** Günlükleme (logging) framework'leri ile hataları kaydedin.

```csharp
var logger = LoggerFactory.Create(builder => builder.AddConsole()).CreateLogger("ErrorLogger");

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

## 5. Global Exception Handler

❌ **Yanlış Kullanım:** Tüm hataları global olarak yönetmemek.

```csharp
app.MapGet("/", () => throw new Exception("Bir hata oluştu!"));
```

✅ **İdeal Kullanım:** Tüm uygulama genelinde hataları ele almak için bir middleware kullanın.

```csharp
app.UseExceptionHandler("/error");

app.Map("/error", (HttpContext context) =>
{
    var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
    return Results.Problem(detail: exception?.Message, statusCode: 500);
});
```

---

## 6. Hata Yönetiminde Özel Sınıflar Kullanımı

❌ **Yanlış Kullanım:** Hata yönetimi için standart `Exception` sınıfını doğrudan kullanmak.

```csharp
throw new Exception("Hatalı işlem");
```

✅ **İdeal Kullanım:** Özel hata sınıfları oluşturarak daha anlamlı hata mesajları sağlayın.

```csharp
public class CustomException : Exception
{
    public CustomException(string message) : base(message) { }
}

throw new CustomException("Bu özel bir hatadır.");
```

---

## 7. Hataların Sessizce Yutulması

❌ **Yanlış Kullanım:** Hataları yakalayıp hiçbir işlem yapmamak.

```csharp
try
{
    // İşlem
}
catch
{
    // Sessizce yutulan hata
}
```

✅ **İdeal Kullanım:** Hataları doğru bir şekilde işleyin veya kaydedin.

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