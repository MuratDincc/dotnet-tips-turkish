# Observer Pattern

Observer deseni, nesneler arasında bire-çok bağımlılık kurarak olay tabanlı iletişim sağlar; yanlış kullanımlar bellek sızıntısı ve sıkı bağımlılık sorunlarına yol açar.

---

## 1. Event Handler ile Bellek Sızıntısı

❌ **Yanlış Kullanım:** Event handler'ları unsubscribe etmemek.

```csharp
public class OrderPage
{
    public OrderPage(OrderService service)
    {
        service.OrderCompleted += OnOrderCompleted; // Unsubscribe yapılmıyor
    }

    private void OnOrderCompleted(object sender, OrderEventArgs e)
    {
        // UI güncelle
    }
}
```

✅ **İdeal Kullanım:** `IDisposable` ile event handler'ı temizleyin.

```csharp
public class OrderPage : IDisposable
{
    private readonly OrderService _service;

    public OrderPage(OrderService service)
    {
        _service = service;
        _service.OrderCompleted += OnOrderCompleted;
    }

    private void OnOrderCompleted(object sender, OrderEventArgs e)
    {
        // UI güncelle
    }

    public void Dispose()
    {
        _service.OrderCompleted -= OnOrderCompleted;
    }
}
```

---

## 2. Sıkı Bağımlı Observer Yapısı

❌ **Yanlış Kullanım:** Observer'ı concrete sınıfa bağlamak.

```csharp
public class StockService
{
    private readonly EmailService _emailService;
    private readonly SmsService _smsService;

    public void UpdatePrice(decimal price)
    {
        _emailService.Send($"Yeni fiyat: {price}");
        _smsService.Send($"Yeni fiyat: {price}");
    }
}
```

✅ **İdeal Kullanım:** Interface tabanlı observer listesi ile gevşek bağlılık sağlayın.

```csharp
public interface IPriceObserver
{
    Task OnPriceChanged(decimal newPrice);
}

public class StockService
{
    private readonly IEnumerable<IPriceObserver> _observers;

    public StockService(IEnumerable<IPriceObserver> observers)
    {
        _observers = observers;
    }

    public async Task UpdatePrice(decimal price)
    {
        foreach (var observer in _observers)
            await observer.OnPriceChanged(price);
    }
}
```

---

## 3. MediatR Notification Kullanmamak

❌ **Yanlış Kullanım:** Manuel event dispatch mekanizması yazmak.

```csharp
public class OrderService
{
    private readonly List<Action<Order>> _handlers = new();

    public void Subscribe(Action<Order> handler) => _handlers.Add(handler);

    public void Complete(Order order)
    {
        // İş mantığı
        foreach (var handler in _handlers)
            handler(order);
    }
}
```

✅ **İdeal Kullanım:** MediatR Notification ile event yayınlayın.

```csharp
public record OrderCompletedEvent(Order Order) : INotification;

public class OrderService
{
    private readonly IMediator _mediator;

    public OrderService(IMediator mediator) => _mediator = mediator;

    public async Task Complete(Order order)
    {
        // İş mantığı
        await _mediator.Publish(new OrderCompletedEvent(order));
    }
}

public class SendEmailHandler : INotificationHandler<OrderCompletedEvent>
{
    public Task Handle(OrderCompletedEvent notification, CancellationToken ct)
    {
        // E-posta gönder
        return Task.CompletedTask;
    }
}
```

---

## 4. Senkron Observer ile Performans Sorunu

❌ **Yanlış Kullanım:** Tüm observer'ları senkron çalıştırmak.

```csharp
public void Notify(OrderEvent orderEvent)
{
    foreach (var observer in _observers)
    {
        observer.Handle(orderEvent); // Uzun süren observer tüm zinciri bloklar
    }
}
```

✅ **İdeal Kullanım:** Asenkron ve paralel çalıştırın.

```csharp
public async Task NotifyAsync(OrderEvent orderEvent)
{
    var tasks = _observers.Select(o => o.HandleAsync(orderEvent));
    await Task.WhenAll(tasks);
}
```

---

## 5. Observer'da Exception Yönetimi

❌ **Yanlış Kullanım:** Bir observer'ın hatası tüm zinciri kırmak.

```csharp
public async Task NotifyAsync(OrderEvent orderEvent)
{
    foreach (var observer in _observers)
    {
        await observer.HandleAsync(orderEvent); // Birisi hata fırlatırsa geri kalanı çalışmaz
    }
}
```

✅ **İdeal Kullanım:** Her observer'ı izole ederek hata yönetimi yapın.

```csharp
public async Task NotifyAsync(OrderEvent orderEvent)
{
    foreach (var observer in _observers)
    {
        try
        {
            await observer.HandleAsync(orderEvent);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "{Observer} işlenirken hata oluştu", observer.GetType().Name);
        }
    }
}
```