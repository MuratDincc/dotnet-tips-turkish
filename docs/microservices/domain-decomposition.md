# Domain Decomposition

Domain Decomposition, monolitik yapıyı doğru sınırlarla microservice'lere ayırmayı sağlar; yanlış ayrıştırma veri tutarsızlığına ve servisler arası aşırı bağımlılığa yol açar.

---

## 1. Teknik Katmana Göre Ayrıştırma

❌ **Yanlış Kullanım:** Servisleri teknik katmanlara göre bölmek.

```csharp
// API Service - tüm controller'lar burada
// Business Service - tüm iş mantığı burada
// Data Service - tüm veri erişimi burada
// Her özellik değişikliği 3 servisi birden etkiler
```

✅ **İdeal Kullanım:** İş alanına (bounded context) göre ayrıştırın.

```csharp
// Order Service - sipariş ile ilgili her şey
public class OrderService
{
    // Controller, Business Logic, Data Access - tek servis içinde
    [HttpPost("api/orders")]
    public async Task<IActionResult> Create(CreateOrderDto dto) { /* ... */ }
}

// Payment Service - ödeme ile ilgili her şey
// Inventory Service - stok ile ilgili her şey
// Shipping Service - kargo ile ilgili her şey
```

---

## 2. Shared Database Kullanmak

❌ **Yanlış Kullanım:** Birden fazla servisin aynı veritabanını paylaşması.

```csharp
// Order Service
public class OrderRepository
{
    public async Task CreateAsync(Order order)
    {
        await _sharedDb.Orders.AddAsync(order);
        var product = await _sharedDb.Products.FindAsync(order.ProductId); // Products tablosu Inventory'nin!
        product.Stock -= order.Quantity;
        await _sharedDb.SaveChangesAsync();
    }
}
```

✅ **İdeal Kullanım:** Her servisin kendi veritabanı olsun, event ile senkronize edilsin.

```csharp
// Order Service - kendi DB'si
public class OrderRepository
{
    private readonly OrderDbContext _context; // Sadece Orders tablosu

    public async Task CreateAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
        await _context.SaveChangesAsync();
    }
}

// Inventory Service - kendi DB'si, event ile dinler
public class OrderCreatedHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly InventoryDbContext _context;

    public async Task Handle(OrderCreatedEvent notification, CancellationToken ct)
    {
        var product = await _context.Products.FindAsync(notification.ProductId);
        product.ReserveStock(notification.Quantity);
        await _context.SaveChangesAsync(ct);
    }
}
```

---

## 3. Bounded Context Sınırlarını Yanlış Belirlemek

❌ **Yanlış Kullanım:** Çok ince granülaritede servisler oluşturmak.

```csharp
// Ayrı servis: AddressService
// Ayrı servis: PhoneService
// Ayrı servis: EmailService
// Ayrı servis: CustomerProfileService
// Bir müşteri bilgisi güncellemek için 4 servise çağrı gerekir
```

✅ **İdeal Kullanım:** İlişkili kavramları aynı bounded context'te tutun.

```csharp
// Customer Service - müşteri ile ilgili tüm kavramlar burada
public class Customer
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public Address Address { get; private set; }      // Value Object
    public PhoneNumber Phone { get; private set; }     // Value Object
    public Email Email { get; private set; }           // Value Object

    public void UpdateProfile(string name, Address address, PhoneNumber phone, Email email)
    {
        Name = name;
        Address = address;
        Phone = phone;
        Email = email;
        AddDomainEvent(new CustomerProfileUpdated(Id));
    }
}
```

---

## 4. Shared Kernel'ı Yanlış Kullanmak

❌ **Yanlış Kullanım:** Büyük shared library ile servisler arası sıkı bağlılık.

```csharp
// SharedLibrary.dll - tüm servisler buna bağımlı
public class Order { /* ... */ }
public class Customer { /* ... */ }
public class Product { /* ... */ }
public class Payment { /* ... */ }
public static class Utils { /* ... */ }
public static class Constants { /* ... */ }
// Bir değişiklik tüm servislerin deploy edilmesini gerektirir
```

✅ **İdeal Kullanım:** Shared Kernel'ı minimal tutun, sadece ortak kontratları paylaşın.

```csharp
// SharedKernel NuGet paketi - minimal
public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

public abstract class Entity
{
    public int Id { get; protected set; }
}

public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetEqualityComponents();
}

// Integration Events - servisler arası kontratlar
public record OrderCreatedIntegrationEvent(int OrderId, int CustomerId, decimal Total) : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}
```

---

## 5. Anti-Corruption Layer Kullanmamak

❌ **Yanlış Kullanım:** Harici servisin modelini doğrudan kullanmak.

```csharp
public class OrderService
{
    public async Task CreateAsync(ExternalOrderDto externalOrder)
    {
        // Harici sistemin modeli doğrudan kullanılıyor
        var order = new Order
        {
            Id = externalOrder.order_id,         // snake_case
            Total = externalOrder.total_amount,   // Farklı alan adı
            Status = MapStatus(externalOrder.sts) // Kısaltılmış alan
        };
    }
}
```

✅ **İdeal Kullanım:** Anti-Corruption Layer ile harici modeli kendi modeline dönüştürün.

```csharp
public interface IExternalOrderAdapter
{
    Task<Order> TranslateAsync(ExternalOrderDto externalOrder);
}

public class ExternalOrderAdapter : IExternalOrderAdapter
{
    public Task<Order> TranslateAsync(ExternalOrderDto externalOrder)
    {
        var order = Order.Create(
            customerId: externalOrder.customer_ref,
            total: new Money(externalOrder.total_amount, "TRY"),
            status: MapStatus(externalOrder.sts)
        );
        return Task.FromResult(order);
    }

    private OrderStatus MapStatus(string externalStatus) => externalStatus switch
    {
        "NEW" => OrderStatus.Pending,
        "PAID" => OrderStatus.Confirmed,
        "SENT" => OrderStatus.Shipped,
        _ => OrderStatus.Unknown
    };
}

public class OrderService
{
    private readonly IExternalOrderAdapter _adapter;

    public async Task ImportAsync(ExternalOrderDto externalOrder)
    {
        var order = await _adapter.TranslateAsync(externalOrder);
        await _repository.AddAsync(order);
    }
}
```