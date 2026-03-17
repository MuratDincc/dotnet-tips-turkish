# File-Scoped Namespaces

C# 10.0 ile gelen file-scoped namespace özelliği, her dosyada yalnızca bir namespace tanımlamanıza olanak tanır ve gereksiz girintiyi ortadan kaldırır. Ancak, bazı durumlarda doğru kullanım önemlidir.

---

## 1. Gereksiz Girinti ile Namespace Kullanımı

❌ **Yanlış Kullanım:** Klasik namespace tanımı ile tüm kodu bir seviye daha girintili yazmak.

```csharp
namespace MyProject.Services
{
    public class ProductService
    {
        public void GetAll()
        {
            // ...
        }
    }
}
```

✅ **İdeal Kullanım:** File-scoped namespace ile girintiyi azaltın.

```csharp
namespace MyProject.Services;

public class ProductService
{
    public void GetAll()
    {
        // ...
    }
}
```

---

## 2. Bir Dosyada Birden Fazla Namespace Tanımlamak

❌ **Yanlış Kullanım:** File-scoped namespace ile aynı dosyada birden fazla namespace tanımlamaya çalışmak.

```csharp
namespace MyProject.Models;
namespace MyProject.DTOs; // Derleme hatası!

public class ProductDto { }
```

✅ **İdeal Kullanım:** Her dosyada yalnızca bir namespace kullanın. Farklı namespace'ler için ayrı dosyalar oluşturun.

```csharp
// Models/Product.cs
namespace MyProject.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

```csharp
// DTOs/ProductDto.cs
namespace MyProject.DTOs;

public class ProductDto
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

---

## 3. Klasik ve File-Scoped Namespace'i Karıştırmak

❌ **Yanlış Kullanım:** Aynı projede bazı dosyalarda klasik, bazılarında file-scoped namespace kullanmak.

```csharp
// ProductService.cs
namespace MyProject.Services;

public class ProductService { }
```

```csharp
// OrderService.cs
namespace MyProject.Services
{
    public class OrderService { }
}
```

✅ **İdeal Kullanım:** Proje genelinde tutarlı bir stil tercih edin. `.editorconfig` ile bunu zorunlu hale getirin.

```ini
# .editorconfig
[*.cs]
csharp_style_namespace_declarations = file_scoped:warning
```

```csharp
// ProductService.cs
namespace MyProject.Services;

public class ProductService { }
```

```csharp
// OrderService.cs
namespace MyProject.Services;

public class OrderService { }
```

---

## 4. Nested Sınıflarda Yanlış Kullanım

❌ **Yanlış Kullanım:** File-scoped namespace'in nested sınıfları etkileyeceğini düşünmek.

```csharp
namespace MyProject.Services;

// File-scoped namespace yalnızca dosya seviyesinde çalışır
// Nested sınıflar hâlâ normal şekilde tanımlanır
public class OrderService
{
    // Bu sınıf MyProject.Services.OrderService.OrderItem olarak erişilir
    public class OrderItem
    {
        public string ProductName { get; set; }
        public int Quantity { get; set; }
    }
}
```

✅ **İdeal Kullanım:** Nested sınıflar yerine ayrı dosyalar ve aynı namespace altında bağımsız sınıflar tercih edin.

```csharp
// OrderService.cs
namespace MyProject.Services;

public class OrderService
{
    public void ProcessOrder(OrderItem item)
    {
        // ...
    }
}
```

```csharp
// OrderItem.cs
namespace MyProject.Services;

public class OrderItem
{
    public string ProductName { get; set; }
    public int Quantity { get; set; }
}
```

---

## 5. EditorConfig ile Kod Stili Zorunlu Kılma

Proje genelinde file-scoped namespace kullanımını `.editorconfig` ile zorunlu hale getirebilirsiniz.

```ini
# .editorconfig
[*.cs]
# File-scoped namespace tercih et (suggestion, warning veya error)
csharp_style_namespace_declarations = file_scoped:warning
```

**Severity seviyeleri:**

| Seviye | Açıklama |
|--------|----------|
| `suggestion` | IDE'de öneri olarak gösterilir |
| `warning` | Derleme sırasında uyarı verir |
| `error` | Derleme sırasında hata verir, kod derlenmez |

---

## 6. Mevcut Projeyi Dönüştürme

Mevcut bir projeyi file-scoped namespace'e dönüştürmek için Visual Studio veya `dotnet format` kullanabilirsiniz.

```bash
# dotnet format ile otomatik dönüştürme
dotnet format --diagnostics IDE0161
```

**Not:** Dönüştürme öncesinde `.editorconfig` dosyanızda `csharp_style_namespace_declarations = file_scoped` ayarının tanımlı olduğundan emin olun.