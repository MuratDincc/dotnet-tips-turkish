# Static Files Optimizasyon

Statik dosyalar (CSS, JavaScript, görseller) bir web uygulamasının temel yapı taşlarıdır. Yanlış yapılandırılmış statik dosya yönetimi, uygulama performansını düşürebilir ve kötü bir kullanıcı deneyimine yol açabilir.

---

## 1. Statik Dosyaları Doğrudan Serve Etmek

❌ **Yanlış Kullanım:** Statik dosyaları manuel olarak işlemek.

```csharp
app.MapGet("/static/{filename}", async (HttpContext context, string filename) =>
{
    var filePath = Path.Combine("wwwroot", filename);
    if (File.Exists(filePath))
    {
        await context.Response.SendFileAsync(filePath);
    }
    else
    {
        context.Response.StatusCode = 404;
    }
});
```

✅ **İdeal Kullanım:** `UseStaticFiles` middleware'ini kullanarak statik dosyaları serve edin.

```csharp
app.UseStaticFiles();
```

---

## 2. Gzip veya Brotli Sıkıştırmayı İhmal Etmek

❌ **Yanlış Kullanım:** Sıkıştırma olmadan büyük dosyaları serve etmek.

```plaintext
Tüm dosyalar sıkıştırılmadan gönderiliyor.
```

✅ **İdeal Kullanım:** `ResponseCompression` middleware'ini etkinleştirin.

```csharp
app.UseResponseCompression();

builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});
```

---

## 3. Cache-Control ve ETag Başlıklarının Eksikliği

❌ **Yanlış Kullanım:** Statik dosyalar için önbellekleme başlıklarını belirtmemek.

```plaintext
Cache-Control başlığı olmadan dosyalar gönderiliyor.
```

✅ **İdeal Kullanım:** Önbellekleme başlıklarını yapılandırın.

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = context =>
    {
        context.Context.Response.Headers["Cache-Control"] = "public,max-age=31536000";
        context.Context.Response.Headers["ETag"] = ""unique-id"";
    }
});
```

---

## 4. Yüksek Çözünürlüklü Görsellerin Optimize Edilmemesi

❌ **Yanlış Kullanım:** Büyük ve optimize edilmemiş görselleri serve etmek.

```plaintext
images/background.png (5MB)
```

✅ **İdeal Kullanım:** Görselleri sıkıştırarak ve CDN kullanarak optimize edin.

- **Araçlar:** ImageMagick, TinyPNG
- **Örnek:** Görselleri bir CDN üzerinden dağıtmak.

```plaintext
https://cdn.example.com/images/background.png
```

---

## 5. Güvenlik Başlıklarının Eksikliği

❌ **Yanlış Kullanım:** Statik dosyalar için güvenlik başlıklarını ihmal etmek.

```plaintext
Statik dosyalar X-Content-Type-Options başlığı olmadan gönderiliyor.
```

✅ **İdeal Kullanım:** Güvenlik başlıklarını yapılandırın.

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = context =>
    {
        context.Context.Response.Headers["X-Content-Type-Options"] = "nosniff";
        context.Context.Response.Headers["Content-Security-Policy"] = "default-src 'self'";
    }
});
```

---

## 6. CDN Kullanımının İhmal Edilmesi

❌ **Yanlış Kullanım:** Tüm statik dosyaları doğrudan sunucudan serve etmek.

```plaintext
Tüm dosyalar sunucudan yükleniyor.
```

✅ **İdeal Kullanım:** Statik dosyaları bir CDN ile dağıtın.

```plaintext
https://cdn.example.com/styles/main.css
https://cdn.example.com/scripts/app.js
```

---

## 7. Fazla Sayıda HTTP İsteği

❌ **Yanlış Kullanım:** Fazla sayıda küçük dosyayı ayrı ayrı serve etmek.

```html
<link rel="stylesheet" href="/css/reset.css">
<link rel="stylesheet" href="/css/grid.css">
<link rel="stylesheet" href="/css/theme.css">
```

✅ **İdeal Kullanım:** Dosyaları birleştirerek HTTP isteklerini azaltın.

```html
<link rel="stylesheet" href="/css/styles.bundle.css">
```