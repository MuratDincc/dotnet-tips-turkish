# Routing ve URL Yönetimi

Routing ve URL yönetimi, web uygulamalarının düzgün çalışması ve kullanıcı deneyiminin artırılması için kritik öneme sahiptir. Yanlış yapılandırılmış yönlendirme, performans sorunlarına ve hatalı sonuçlara yol açabilir.

---

## 1. Yanlış Route Tanımlamaları

❌ **Yanlış Kullanım:** Çakışan veya karışıklık yaratan route'lar.

```csharp
app.MapGet("/products", () => "Tüm ürünler");
app.MapGet("/products/{id}", (int id) => $"Ürün ID: {id}");
app.MapGet("/products/{category}", (string category) => $"Kategori: {category}"); // Çakışma
```

✅ **İdeal Kullanım:** Route'ları açıkça tanımlayın ve çakışmaları önleyin.

```csharp
app.MapGet("/products", () => "Tüm ürünler");
app.MapGet("/products/{id:int}", (int id) => $"Ürün ID: {id}");
app.MapGet("/products/category/{category}", (string category) => $"Kategori: {category}");
```

---

## 2. Gereksiz Route Parametreleri

❌ **Yanlış Kullanım:** Route parametrelerini gereksiz yere kullanmak.

```csharp
app.MapGet("/user/{id}/details", (int id) => $"Kullanıcı ID: {id}");
```

✅ **İdeal Kullanım:** Route parametrelerini anlamlı ve minimum düzeyde tutun.

```csharp
app.MapGet("/users/{id}", (int id) => $"Kullanıcı ID: {id}");
```

---

## 3. Catch-All Route Kullanımı

❌ **Yanlış Kullanım:** Catch-all route'ları dikkatsizce kullanmak.

```csharp
app.MapGet("/{*path}", (string path) => $"Path: {path}"); // Diğer route'ları engelleyebilir
```

✅ **İdeal Kullanım:** Catch-all route'ları dikkatlice ve belirli bir amaca yönelik kullanın.

```csharp
app.MapGet("/files/{*filepath}", (string filepath) => $"Dosya yolu: {filepath}");
```

---

## 4. Route Adlarının Kullanılmaması

❌ **Yanlış Kullanım:** Route adlarını belirtmeden URL'lerle çalışmak.

```csharp
app.MapGet("/home", () => "Ana Sayfa");
```

✅ **İdeal Kullanım:** Route adlarını belirterek yönlendirme işlemlerini daha okunabilir hale getirin.

```csharp
app.MapGet("/home", () => "Ana Sayfa").WithName("Home");
```

---

## 5. Query Parametrelerinin Yanlış Kullanımı

❌ **Yanlış Kullanım:** Query parametrelerini manuel olarak işlemek.

```csharp
app.MapGet("/search", (HttpContext context) =>
{
    var query = context.Request.Query["q"];
    return $"Arama: {query}";
});
```

✅ **İdeal Kullanım:** Query parametrelerini doğrudan metot parametresi olarak bağlayın.

```csharp
app.MapGet("/search", (string q) => $"Arama: {q}");
```

---

## 6. Route Constraint Eksikliği

❌ **Yanlış Kullanım:** Route parametrelerini doğrulama yapmadan kullanmak.

```csharp
app.MapGet("/users/{id}", (string id) => $"Kullanıcı ID: {id}");
```

✅ **İdeal Kullanım:** Route constraint'leri kullanarak doğru veri tiplerini belirleyin.

```csharp
app.MapGet("/users/{id:int}", (int id) => $"Kullanıcı ID: {id}");
```

---

## 7. Route Prioritization Sorunları

❌ **Yanlış Kullanım:** Özel route'ları genel route'ların altına yerleştirmek.

```csharp
app.MapGet("/{category}", (string category) => $"Kategori: {category}");
app.MapGet("/products", () => "Ürün listesi"); // Çalışmaz
```

✅ **İdeal Kullanım:** Daha özel route'ları önce tanımlayın.

```csharp
app.MapGet("/products", () => "Ürün listesi");
app.MapGet("/{category}", (string category) => $"Kategori: {category}");
```