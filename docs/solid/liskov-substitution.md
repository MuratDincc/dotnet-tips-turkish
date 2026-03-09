# Liskov Substitution Principle (LSP)

Alt sınıflar, üst sınıfların yerine sorunsuz kullanılabilmelidir; davranış kontratını bozan kalıtım hiyerarşileri beklenmeyen hatalara yol açar.

---

## 1. NotImplementedException Fırlatmak

❌ **Yanlış Kullanım:** Interface metodunu uygulamayıp exception fırlatmak.

```csharp
public interface IFileStorage
{
    Task UploadAsync(Stream file);
    Task<Stream> DownloadAsync(string path);
    Task DeleteAsync(string path);
}

public class ReadOnlyStorage : IFileStorage
{
    public Task UploadAsync(Stream file) => throw new NotImplementedException();
    public Task<Stream> DownloadAsync(string path) { /* ... */ }
    public Task DeleteAsync(string path) => throw new NotImplementedException();
}
```

✅ **İdeal Kullanım:** Interface'leri sorumluluk bazında ayırın.

```csharp
public interface IFileReader
{
    Task<Stream> DownloadAsync(string path);
}

public interface IFileWriter
{
    Task UploadAsync(Stream file);
    Task DeleteAsync(string path);
}

public class ReadOnlyStorage : IFileReader
{
    public Task<Stream> DownloadAsync(string path) { /* ... */ }
}

public class FullStorage : IFileReader, IFileWriter
{
    public Task<Stream> DownloadAsync(string path) { /* ... */ }
    public Task UploadAsync(Stream file) { /* ... */ }
    public Task DeleteAsync(string path) { /* ... */ }
}
```

---

## 2. Precondition'ları Güçlendirmek

❌ **Yanlış Kullanım:** Alt sınıfta üst sınıftan daha katı kurallar koymak.

```csharp
public class Rectangle
{
    public virtual void SetWidth(int width)
    {
        if (width <= 0) throw new ArgumentException();
        Width = width;
    }
}

public class RestrictedRectangle : Rectangle
{
    public override void SetWidth(int width)
    {
        if (width <= 0 || width > 100) // Üst sınıfta olmayan ek kısıtlama
            throw new ArgumentException();
        Width = width;
    }
}
```

✅ **İdeal Kullanım:** Alt sınıf üst sınıfın kontratına sadık kalsın.

```csharp
public class Rectangle
{
    public virtual void SetWidth(int width)
    {
        if (width <= 0) throw new ArgumentException();
        Width = width;
    }
}

public class RestrictedRectangle : Rectangle
{
    public override void SetWidth(int width)
    {
        base.SetWidth(width); // Üst sınıf kontratını korur
        // Ek iş mantığı
    }

    public int MaxWidth => 100;
    public bool IsWithinLimit => Width <= MaxWidth;
}
```

---

## 3. Postcondition'ları Zayıflatmak

❌ **Yanlış Kullanım:** Üst sınıfın garanti ettiği davranışı alt sınıfta bozmak.

```csharp
public class OrderRepository
{
    public virtual async Task<Order> GetByIdAsync(int id)
    {
        return await _context.Orders.FindAsync(id)
            ?? throw new NotFoundException($"Sipariş bulunamadı: {id}");
    }
}

public class CachedOrderRepository : OrderRepository
{
    public override async Task<Order> GetByIdAsync(int id)
    {
        return _cache.Get<Order>(id); // Cache'de yoksa null dönebilir!
    }
}
```

✅ **İdeal Kullanım:** Alt sınıf üst sınıfın garanti ettiği davranışı korusun.

```csharp
public class CachedOrderRepository : OrderRepository
{
    public override async Task<Order> GetByIdAsync(int id)
    {
        var cached = _cache.Get<Order>(id);
        if (cached != null) return cached;

        var order = await base.GetByIdAsync(id); // NotFoundException garantisi korunur
        _cache.Set(id, order);
        return order;
    }
}
```

---

## 4. Rectangle-Square Problemi

❌ **Yanlış Kullanım:** Kare'yi dikdörtgenden türetmek.

```csharp
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        set { base.Width = value; base.Height = value; }
    }
    public override int Height
    {
        set { base.Width = value; base.Height = value; }
    }
}

// Beklenmeyen davranış
Rectangle rect = new Square();
rect.Width = 5;
rect.Height = 3;
// Area 15 beklenirken 9 döner
```

✅ **İdeal Kullanım:** Ortak interface ile soyutlayın, kalıtımdan kaçının.

```csharp
public interface IShape
{
    int Area { get; }
}

public class Rectangle : IShape
{
    public int Width { get; init; }
    public int Height { get; init; }
    public int Area => Width * Height;
}

public class Square : IShape
{
    public int Side { get; init; }
    public int Area => Side * Side;
}
```

---

## 5. Collection Kalıtımında LSP İhlali

❌ **Yanlış Kullanım:** Readonly collection'ı writable collection'dan türetmek.

```csharp
public class ReadOnlyProductList : List<Product>
{
    public new void Add(Product item) => throw new NotSupportedException();
    public new void Remove(Product item) => throw new NotSupportedException();
}

// LSP ihlali: List<Product> bekleyen kod çalışmaz
void AddProducts(List<Product> products)
{
    products.Add(new Product()); // NotSupportedException!
}
```

✅ **İdeal Kullanım:** Doğru soyutlama seviyesinde interface kullanın.

```csharp
public class ProductCatalog
{
    private readonly List<Product> _products = new();

    public IReadOnlyCollection<Product> Products => _products.AsReadOnly();

    public void Add(Product product) => _products.Add(product);
}

// IReadOnlyCollection kullanarak tüketiciye doğru kontratı sunun
void DisplayProducts(IReadOnlyCollection<Product> products)
{
    foreach (var product in products)
        Console.WriteLine(product.Name);
}
```