# Strategy Pattern

Strategy deseni, algoritmaları birbirinin yerine geçebilir hale getirir; yanlış uygulamalar ise if-else zincirlerine ve bakımı zor koda neden olur.

---

## 1. If-Else Zinciri ile Strateji Seçimi

❌ **Yanlış Kullanım:** İş mantığını if-else ile dallandırmak.

```csharp
public class DiscountService
{
    public decimal Calculate(string customerType, decimal amount)
    {
        if (customerType == "premium")
            return amount * 0.20m;
        else if (customerType == "gold")
            return amount * 0.10m;
        else
            return amount * 0.05m;
    }
}
```

✅ **İdeal Kullanım:** Her stratejiyi ayrı sınıf olarak tanımlayın.

```csharp
public interface IDiscountStrategy
{
    decimal Calculate(decimal amount);
}

public class PremiumDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal amount) => amount * 0.20m;
}

public class GoldDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal amount) => amount * 0.10m;
}

public class DiscountService
{
    private readonly IDiscountStrategy _strategy;

    public DiscountService(IDiscountStrategy strategy)
    {
        _strategy = strategy;
    }

    public decimal Calculate(decimal amount) => _strategy.Calculate(amount);
}
```

---

## 2. Runtime'da Strateji Değiştirememe

❌ **Yanlış Kullanım:** Stratejiyi constructor'da sabitlemek.

```csharp
public class ExportService
{
    private readonly CsvExporter _exporter = new CsvExporter();

    public byte[] Export(IEnumerable<Order> orders)
    {
        return _exporter.Export(orders);
    }
}
```

✅ **İdeal Kullanım:** Stratejiyi runtime'da method parametresi olarak alın.

```csharp
public interface IExportStrategy
{
    byte[] Export(IEnumerable<Order> orders);
}

public class ExportService
{
    public byte[] Export(IEnumerable<Order> orders, IExportStrategy strategy)
    {
        return strategy.Export(orders);
    }
}
```

---

## 3. DI ile Strateji Çözümleme

❌ **Yanlış Kullanım:** Strateji seçimini manuel if-else ile yapmak.

```csharp
public class PaymentService
{
    public void Pay(string method, decimal amount)
    {
        if (method == "credit") new CreditCardPayment().Pay(amount);
        else if (method == "bank") new BankTransferPayment().Pay(amount);
    }
}
```

✅ **İdeal Kullanım:** DI container'dan tüm stratejileri inject edip doğru olanı seçin.

```csharp
public interface IPaymentStrategy
{
    string Method { get; }
    void Pay(decimal amount);
}

public class PaymentService
{
    private readonly Dictionary<string, IPaymentStrategy> _strategies;

    public PaymentService(IEnumerable<IPaymentStrategy> strategies)
    {
        _strategies = strategies.ToDictionary(s => s.Method);
    }

    public void Pay(string method, decimal amount)
    {
        if (!_strategies.TryGetValue(method, out var strategy))
            throw new ArgumentException($"Bilinmeyen ödeme yöntemi: {method}");

        strategy.Pay(amount);
    }
}
```

---

## 4. Strateji İçinde Ortak Mantığı Tekrarlamak

❌ **Yanlış Kullanım:** Her strateji sınıfında aynı kodu tekrarlamak.

```csharp
public class EmailSender : INotificationStrategy
{
    public void Send(string message)
    {
        var sanitized = message.Trim().Replace("<", "&lt;");
        // E-posta gönder
    }
}

public class SmsSender : INotificationStrategy
{
    public void Send(string message)
    {
        var sanitized = message.Trim().Replace("<", "&lt;");
        // SMS gönder
    }
}
```

✅ **İdeal Kullanım:** Ortak mantığı base class veya decorator ile yönetin.

```csharp
public abstract class NotificationStrategyBase : INotificationStrategy
{
    public void Send(string message)
    {
        var sanitized = Sanitize(message);
        SendInternal(sanitized);
    }

    protected abstract void SendInternal(string message);

    private string Sanitize(string message) => message.Trim().Replace("<", "&lt;");
}

public class EmailSender : NotificationStrategyBase
{
    protected override void SendInternal(string message)
    {
        // E-posta gönder
    }
}
```

---

## 5. Strateji Seçiminde Magic String Kullanmak

❌ **Yanlış Kullanım:** String literal ile strateji eşleştirmek.

```csharp
var strategy = provider.GetStrategy("credit_card_v2");
```

✅ **İdeal Kullanım:** Enum veya strongly-typed tanımlayıcılar kullanın.

```csharp
public enum PaymentMethod
{
    CreditCard,
    BankTransfer,
    Wallet
}

public interface IPaymentStrategy
{
    PaymentMethod Method { get; }
    Task ProcessAsync(decimal amount);
}

public class PaymentResolver
{
    private readonly Dictionary<PaymentMethod, IPaymentStrategy> _strategies;

    public PaymentResolver(IEnumerable<IPaymentStrategy> strategies)
    {
        _strategies = strategies.ToDictionary(s => s.Method);
    }

    public IPaymentStrategy Resolve(PaymentMethod method) => _strategies[method];
}
```