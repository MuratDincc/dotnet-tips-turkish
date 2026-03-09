# Open/Closed Principle (OCP)

Sınıflar genişlemeye açık, değişikliğe kapalı olmalıdır; yeni gereksinimler mevcut kodu değiştirmeden eklenebilmelidir.

---

## 1. Switch-Case ile Genişleme

❌ **Yanlış Kullanım:** Her yeni tip eklendiğinde mevcut kodu değiştirmek.

```csharp
public class ShippingCostCalculator
{
    public decimal Calculate(string method, decimal weight)
    {
        switch (method)
        {
            case "standard": return weight * 5;
            case "express": return weight * 10;
            case "overnight": return weight * 20;
            // Yeni kargo yöntemi eklemek için bu sınıfı değiştirmek gerekir
            default: throw new ArgumentException("Bilinmeyen yöntem");
        }
    }
}
```

✅ **İdeal Kullanım:** Polymorphism ile genişlemeye açık yapı kurun.

```csharp
public interface IShippingCostStrategy
{
    string Method { get; }
    decimal Calculate(decimal weight);
}

public class StandardShipping : IShippingCostStrategy
{
    public string Method => "standard";
    public decimal Calculate(decimal weight) => weight * 5;
}

public class ExpressShipping : IShippingCostStrategy
{
    public string Method => "express";
    public decimal Calculate(decimal weight) => weight * 10;
}

public class ShippingCostCalculator
{
    private readonly Dictionary<string, IShippingCostStrategy> _strategies;

    public ShippingCostCalculator(IEnumerable<IShippingCostStrategy> strategies)
    {
        _strategies = strategies.ToDictionary(s => s.Method);
    }

    public decimal Calculate(string method, decimal weight)
        => _strategies[method].Calculate(weight);
}
```

---

## 2. Doğrudan Sınıf Değiştirme

❌ **Yanlış Kullanım:** Mevcut sınıfa yeni davranış eklemek için kodu değiştirmek.

```csharp
public class NotificationService
{
    public void Send(string type, string message, string recipient)
    {
        if (type == "email")
        {
            // E-posta gönder
        }
        else if (type == "sms")
        {
            // SMS gönder
        }
        // Push notification eklemek için burayı değiştirmek gerekir
    }
}
```

✅ **İdeal Kullanım:** Yeni bildirim kanalını ayrı sınıf olarak ekleyin.

```csharp
public interface INotificationChannel
{
    Task SendAsync(string message, string recipient);
}

public class EmailChannel : INotificationChannel
{
    public Task SendAsync(string message, string recipient) { /* ... */ }
}

public class SmsChannel : INotificationChannel
{
    public Task SendAsync(string message, string recipient) { /* ... */ }
}

// Yeni kanal eklemek için sadece yeni sınıf oluşturulur
public class PushChannel : INotificationChannel
{
    public Task SendAsync(string message, string recipient) { /* ... */ }
}

// DI'da tüm kanalları kaydedin
builder.Services.AddScoped<INotificationChannel, EmailChannel>();
builder.Services.AddScoped<INotificationChannel, SmsChannel>();
builder.Services.AddScoped<INotificationChannel, PushChannel>();
```

---

## 3. Validation Kurallarını Sınıf İçine Gömmek

❌ **Yanlış Kullanım:** Validasyon kurallarını sabit kodlamak.

```csharp
public class OrderValidator
{
    public bool Validate(Order order)
    {
        if (order.Items.Count == 0) return false;
        if (order.TotalPrice <= 0) return false;
        if (order.TotalPrice > 10000) return false;
        // Yeni kural eklemek için bu metodu değiştirmek gerekir
        return true;
    }
}
```

✅ **İdeal Kullanım:** Kural tabanlı validasyon ile genişletilebilir yapı kurun.

```csharp
public interface IOrderRule
{
    bool IsSatisfiedBy(Order order);
    string ErrorMessage { get; }
}

public class OrderMustHaveItems : IOrderRule
{
    public bool IsSatisfiedBy(Order order) => order.Items.Count > 0;
    public string ErrorMessage => "Sipariş en az bir ürün içermelidir.";
}

public class OrderMaxAmountRule : IOrderRule
{
    public bool IsSatisfiedBy(Order order) => order.TotalPrice <= 10000;
    public string ErrorMessage => "Sipariş tutarı 10.000 TL'yi geçemez.";
}

public class OrderValidator
{
    private readonly IEnumerable<IOrderRule> _rules;

    public OrderValidator(IEnumerable<IOrderRule> rules) => _rules = rules;

    public (bool IsValid, List<string> Errors) Validate(Order order)
    {
        var errors = _rules
            .Where(r => !r.IsSatisfiedBy(order))
            .Select(r => r.ErrorMessage)
            .ToList();

        return (errors.Count == 0, errors);
    }
}
```

---

## 4. Middleware Pipeline ile OCP

❌ **Yanlış Kullanım:** Request işleme mantığını tek bir metoda toplamak.

```csharp
public async Task HandleRequest(HttpContext context)
{
    // Logging
    _logger.LogInformation("Request: {Path}", context.Request.Path);
    // Authentication
    if (!context.User.Identity.IsAuthenticated) { context.Response.StatusCode = 401; return; }
    // Rate Limiting
    if (IsRateLimited(context)) { context.Response.StatusCode = 429; return; }
    // İş mantığı
    await ProcessAsync(context);
}
```

✅ **İdeal Kullanım:** Middleware pipeline ile her concern'ü ayrı katman olarak ekleyin.

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
app.UseAuthentication();
app.UseMiddleware<RateLimitingMiddleware>();
app.UseAuthorization();
app.MapControllers();

// Yeni middleware eklemek mevcut kodu değiştirmez
app.UseMiddleware<CorrelationIdMiddleware>();
```

---

## 5. Decorator ile OCP

❌ **Yanlış Kullanım:** Mevcut servise loglama eklemek için kodu değiştirmek.

```csharp
public class PaymentService : IPaymentService
{
    public async Task<PaymentResult> ProcessAsync(Payment payment)
    {
        _logger.LogInformation("Ödeme başlatıldı"); // Yeni eklendi - mevcut kod değişti
        var result = await _gateway.ChargeAsync(payment);
        _logger.LogInformation("Ödeme tamamlandı"); // Yeni eklendi
        return result;
    }
}
```

✅ **İdeal Kullanım:** Decorator ile mevcut servisi değiştirmeden davranış ekleyin.

```csharp
public class PaymentService : IPaymentService
{
    public async Task<PaymentResult> ProcessAsync(Payment payment)
    {
        return await _gateway.ChargeAsync(payment); // Değişmedi
    }
}

public class LoggingPaymentDecorator : IPaymentService
{
    private readonly IPaymentService _inner;
    private readonly ILogger _logger;

    public LoggingPaymentDecorator(IPaymentService inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessAsync(Payment payment)
    {
        _logger.LogInformation("Ödeme başlatıldı");
        var result = await _inner.ProcessAsync(payment);
        _logger.LogInformation("Ödeme tamamlandı");
        return result;
    }
}
```