# Globalization ve Localization

Globalization ve localization, uygulamanızın farklı diller ve kültürler için uygun hale getirilmesi sürecidir. Yanlış uygulamalar kullanıcı deneyimini olumsuz etkileyebilir veya yanlış dil/format gösterimine neden olabilir.

---

## 1. Sabit Kodlanmış (Hardcoded) Metinler Kullanmak

❌ **Yanlış Kullanım:** Metinleri doğrudan sabit kodlamak.

```csharp
public string GetWelcomeMessage()
{
    return "Welcome to our application!";
}
```

✅ **İdeal Kullanım:** Kaynak dosyalarını kullanarak metinleri yerelleştirin.

**Resources/Texts.resx:**
```xml
<data name="WelcomeMessage" xml:space="preserve">
  <value>Welcome to our application!</value>
</data>
```

**Kullanım:**
```csharp
public string GetWelcomeMessage()
{
    return Resources.Texts.WelcomeMessage;
}
```

---

## 2. Tarih ve Saat Formatlarını Sabit Kodlamak

❌ **Yanlış Kullanım:** Tarih ve saat formatlarını manuel olarak ayarlamak.

```csharp
var date = DateTime.Now.ToString("MM/dd/yyyy");
```

✅ **İdeal Kullanım:** Kültür bilgilerini kullanarak tarih ve saat formatlarını otomatik hale getirin.

```csharp
var date = DateTime.Now.ToString(CultureInfo.CurrentCulture);
```

---

## 3. `Thread.CurrentThread.CurrentCulture`'ı Doğrudan Değiştirmek

❌ **Yanlış Kullanım:** `Thread.CurrentThread.CurrentCulture`'ı manuel olarak değiştirmek.

```csharp
Thread.CurrentThread.CurrentCulture = new CultureInfo("fr-FR");
```

✅ **İdeal Kullanım:** Middleware kullanarak kültür ayarlarını yönetin.

```csharp
app.UseRequestLocalization(new RequestLocalizationOptions
{
    DefaultRequestCulture = new RequestCulture("en-US"),
    SupportedCultures = new[] { new CultureInfo("en-US"), new CultureInfo("fr-FR") },
    SupportedUICultures = new[] { new CultureInfo("en-US"), new CultureInfo("fr-FR") }
});
```

---

## 4. Kullanıcı Tercihlerine Göre Dil Ayarı Yapmamak

❌ **Yanlış Kullanım:** Varsayılan dil ayarını tüm kullanıcılara uygulamak.

```csharp
var culture = new CultureInfo("en-US");
CultureInfo.DefaultThreadCurrentCulture = culture;
CultureInfo.DefaultThreadCurrentUICulture = culture;
```

✅ **İdeal Kullanım:** Kullanıcının tercih ettiği dili dikkate alın.

```csharp
app.Use(async (context, next) =>
{
    var userLanguage = context.Request.Headers["Accept-Language"].ToString();
    var culture = new CultureInfo(userLanguage);
    CultureInfo.CurrentCulture = culture;
    CultureInfo.CurrentUICulture = culture;

    await next();
});
```

---

## 5. Çevirilerin Test Edilmemesi

❌ **Yanlış Kullanım:** Çevirilerin farklı dillerde nasıl görüneceğini test etmemek.

```plaintext
Test edilmeden yerelleştirme yapılır.
```

✅ **İdeal Kullanım:** Farklı dillerde çevirileri test edin.

- Çevirileri test etmek için Visual Studio'da "Set as Startup Culture" özelliğini kullanabilirsiniz.
- Ayrıca `CultureInfo`'yu manuel olarak değiştirebilirsiniz:

```csharp
var culture = new CultureInfo("fr-FR");
CultureInfo.CurrentCulture = culture;
CultureInfo.CurrentUICulture = culture;
```

---

## 6. Veritabanında Sabit Kodlanmış Datalar Kullanmak

❌ **Yanlış Kullanım:** Veritabanında sadece bir dilde içerik saklamak.

```plaintext
ProductName: "Laptop"
```

✅ **İdeal Kullanım:** Veritabanında çoklu dil desteği sağlayın.

```plaintext
ProductName_en: "Laptop"
ProductName_fr: "Ordinateur portable"
```

---

## 7. Yerelleştirilmiş Kaynakların Performansını İzlememek

❌ **Yanlış Kullanım:** Yerelleştirilmiş kaynakların yüklenme performansını göz ardı etmek.

✅ **İdeal Kullanım:** Performans izleme araçları kullanarak kaynakların yüklenme hızını analiz edin.

- **Örnek:** Application Insights, Prometheus

```csharp
var startTime = Stopwatch.StartNew();
var message = Resources.Texts.WelcomeMessage;
startTime.Stop();
logger.LogInformation("Yerelleştirilmiş kaynak {TimeTaken} ms'de yüklendi.", startTime.ElapsedMilliseconds);
```