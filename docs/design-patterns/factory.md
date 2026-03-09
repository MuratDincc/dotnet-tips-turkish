# Factory Pattern

Factory deseni, nesne oluşturma mantığını soyutlayarak kodun esnekliğini artırır; yanlış kullanımlarda ise gereksiz karmaşıklık ve bakım zorluğu ortaya çıkar.

---

## 1. Switch-Case ile Nesne Oluşturma

❌ **Yanlış Kullanım:** Her yeni tip eklendiğinde factory sınıfını değiştirmek.

```csharp
public class NotificationFactory
{
    public INotification Create(string type)
    {
        switch (type)
        {
            case "email": return new EmailNotification();
            case "sms": return new SmsNotification();
            case "push": return new PushNotification();
            default: throw new ArgumentException("Bilinmeyen tip");
        }
    }
}
```

✅ **İdeal Kullanım:** Dictionary tabanlı kayıt sistemi ile OCP'ye uygun factory oluşturun.

```csharp
public class NotificationFactory
{
    private readonly Dictionary<string, Func<INotification>> _creators = new();

    public void Register(string type, Func<INotification> creator)
    {
        _creators[type] = creator;
    }

    public INotification Create(string type)
    {
        if (!_creators.TryGetValue(type, out var creator))
            throw new ArgumentException($"Bilinmeyen tip: {type}");

        return creator();
    }
}
```

---

## 2. Factory İçinde Doğrudan Bağımlılık Oluşturma

❌ **Yanlış Kullanım:** Factory içinde bağımlılıkları `new` ile oluşturmak.

```csharp
public class PaymentProcessorFactory
{
    public IPaymentProcessor Create(string provider)
    {
        return provider switch
        {
            "stripe" => new StripeProcessor(new HttpClient(), new Logger()),
            "paypal" => new PayPalProcessor(new HttpClient(), new Logger()),
            _ => throw new ArgumentException("Bilinmeyen provider")
        };
    }
}
```

✅ **İdeal Kullanım:** DI container ile factory entegrasyonu yapın.

```csharp
public class PaymentProcessorFactory
{
    private readonly IServiceProvider _serviceProvider;

    public PaymentProcessorFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IPaymentProcessor Create(string provider)
    {
        return provider switch
        {
            "stripe" => _serviceProvider.GetRequiredService<StripeProcessor>(),
            "paypal" => _serviceProvider.GetRequiredService<PayPalProcessor>(),
            _ => throw new ArgumentException("Bilinmeyen provider")
        };
    }
}
```

---

## 3. Generic Factory Kullanmamak

❌ **Yanlış Kullanım:** Her entity için ayrı factory sınıfı yazmak.

```csharp
public class OrderValidatorFactory
{
    public IValidator<Order> Create() => new OrderValidator();
}

public class ProductValidatorFactory
{
    public IValidator<Product> Create() => new ProductValidator();
}
```

✅ **İdeal Kullanım:** Generic factory ile tekrarı ortadan kaldırın.

```csharp
public interface IValidatorFactory
{
    IValidator<T> Create<T>();
}

public class ValidatorFactory : IValidatorFactory
{
    private readonly IServiceProvider _serviceProvider;

    public ValidatorFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IValidator<T> Create<T>()
    {
        return _serviceProvider.GetRequiredService<IValidator<T>>();
    }
}
```

---

## 4. Abstract Factory Yerine If-Else Zinciri

❌ **Yanlış Kullanım:** İlişkili nesneleri if-else ile oluşturmak.

```csharp
public class UIComponentFactory
{
    public IButton CreateButton(string theme)
    {
        if (theme == "dark") return new DarkButton();
        else return new LightButton();
    }

    public ITextBox CreateTextBox(string theme)
    {
        if (theme == "dark") return new DarkTextBox();
        else return new LightTextBox();
    }
}
```

✅ **İdeal Kullanım:** Abstract Factory ile tutarlı nesne aileleri oluşturun.

```csharp
public interface IThemeFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
}

public class DarkThemeFactory : IThemeFactory
{
    public IButton CreateButton() => new DarkButton();
    public ITextBox CreateTextBox() => new DarkTextBox();
}

public class LightThemeFactory : IThemeFactory
{
    public IButton CreateButton() => new LightButton();
    public ITextBox CreateTextBox() => new LightTextBox();
}
```

---

## 5. Factory Method'u Interface ile Soyutlamamak

❌ **Yanlış Kullanım:** Concrete factory sınıfına doğrudan bağımlılık.

```csharp
public class OrderService
{
    private readonly ShippingFactory _factory = new ShippingFactory();

    public void Ship(Order order)
    {
        var shipper = _factory.Create(order.ShippingMethod);
        shipper.Ship(order);
    }
}
```

✅ **İdeal Kullanım:** Factory'yi interface ile soyutlayarak bağımlılığı gevşetin.

```csharp
public interface IShippingFactory
{
    IShipper Create(string method);
}

public class OrderService
{
    private readonly IShippingFactory _factory;

    public OrderService(IShippingFactory factory)
    {
        _factory = factory;
    }

    public void Ship(Order order)
    {
        var shipper = _factory.Create(order.ShippingMethod);
        shipper.Ship(order);
    }
}
```

---

## 6. Keyed Services Kullanmamak (.NET 8+)

❌ **Yanlış Kullanım:** Aynı interface'in farklı implementasyonları için manuel factory yazmak.

```csharp
public class StorageFactory
{
    private readonly IServiceProvider _sp;

    public StorageFactory(IServiceProvider sp) => _sp = sp;

    public IStorage Create(string type) => type switch
    {
        "blob" => _sp.GetRequiredService<BlobStorage>(),
        "local" => _sp.GetRequiredService<LocalStorage>(),
        _ => throw new ArgumentException("Bilinmeyen tip")
    };
}
```

✅ **İdeal Kullanım:** .NET 8 Keyed Services ile doğrudan DI desteği kullanın.

```csharp
builder.Services.AddKeyedSingleton<IStorage, BlobStorage>("blob");
builder.Services.AddKeyedSingleton<IStorage, LocalStorage>("local");

public class FileService
{
    private readonly IStorage _storage;

    public FileService([FromKeyedServices("blob")] IStorage storage)
    {
        _storage = storage;
    }
}
```