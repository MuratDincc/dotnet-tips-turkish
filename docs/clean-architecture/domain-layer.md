# Domain Layer

Domain katmanı uygulamanın kalbini oluşturur ve iş kurallarını barındırır; yanlış tasarımlar anemic model ve dışa bağımlı domain yapılarına yol açar.

---

## 1. Anemic Domain Model

❌ **Yanlış Kullanım:** Entity'leri sadece property torbası olarak tasarlamak.

```csharp
public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; } = new();
    public decimal TotalPrice { get; set; }
    public OrderStatus Status { get; set; }
}

// İş mantığı servis katmanına dağılır
public class OrderService
{
    public void AddItem(Order order, Product product, int quantity)
    {
        order.Items.Add(new OrderItem { ProductId = product.Id, Quantity = quantity });
        order.TotalPrice = order.Items.Sum(i => i.Price * i.Quantity);
    }

    public void Cancel(Order order)
    {
        if (order.Status == OrderStatus.Shipped) throw new Exception("Kargolanmış sipariş iptal edilemez");
        order.Status = OrderStatus.Cancelled;
    }
}
```

✅ **İdeal Kullanım:** Rich Domain Model ile iş mantığını entity içinde barındırın.

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();

    public int Id { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public decimal TotalPrice { get; private set; }
    public OrderStatus Status { get; private set; }

    public void AddItem(Product product, int quantity)
    {
        var item = new OrderItem(product.Id, product.Price, quantity);
        _items.Add(item);
        RecalculateTotal();
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Kargolanmış sipariş iptal edilemez.");

        Status = OrderStatus.Cancelled;
    }

    private void RecalculateTotal()
    {
        TotalPrice = _items.Sum(i => i.Price * i.Quantity);
    }
}
```

---

## 2. Value Object Kullanmamak

❌ **Yanlış Kullanım:** Primitive obsession ile değer nesnelerini string/int olarak tutmak.

```csharp
public class Customer
{
    public string Email { get; set; }     // Herhangi bir string olabilir
    public string PhoneNumber { get; set; } // Format kontrolü yok
    public decimal Balance { get; set; }   // Negatif olabilir
}
```

✅ **İdeal Kullanım:** Value Object ile iş kurallarını değer düzeyinde uygulayın.

```csharp
public record Email
{
    public string Value { get; }

    public Email(string value)
    {
        if (!value.Contains('@'))
            throw new DomainException("Geçersiz e-posta adresi.");
        Value = value;
    }
}

public record Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Tutar negatif olamaz.");
        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new DomainException("Para birimleri uyuşmuyor.");
        return new Money(Amount + other.Amount, Currency);
    }
}

public class Customer
{
    public Email Email { get; private set; }
    public Money Balance { get; private set; }
}
```

---

## 3. Domain Event Kullanmamak

❌ **Yanlış Kullanım:** Yan etkileri doğrudan entity içinden çağırmak.

```csharp
public class Order
{
    private readonly IEmailService _emailService;
    private readonly IInventoryService _inventoryService;

    public void Complete()
    {
        Status = OrderStatus.Completed;
        _emailService.SendOrderConfirmation(this);    // Domain altyapıya bağımlı
        _inventoryService.ReduceStock(this.Items);
    }
}
```

✅ **İdeal Kullanım:** Domain Event ile yan etkileri ayırın.

```csharp
public abstract class Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    protected void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

public class Order : Entity
{
    public void Complete()
    {
        Status = OrderStatus.Completed;
        AddDomainEvent(new OrderCompletedEvent(Id, Items));
    }
}

public class OrderCompletedHandler : INotificationHandler<OrderCompletedEvent>
{
    public async Task Handle(OrderCompletedEvent notification, CancellationToken ct)
    {
        // E-posta gönder, stok azalt, vb.
    }
}
```

---

## 4. Domain Katmanında Framework Bağımlılığı

❌ **Yanlış Kullanım:** Domain entity'lerinde EF Core attribute'ları kullanmak.

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class Product
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }

    [Required]
    [MaxLength(200)]
    public string Name { get; set; }

    [Column(TypeName = "decimal(18,2)")]
    public decimal Price { get; set; }
}
```

✅ **İdeal Kullanım:** Domain entity'lerini saf tutun, mapping'i Infrastructure'da yapın.

```csharp
// Domain Layer
public class Product
{
    public int Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }

    public Product(string name, decimal price)
    {
        if (string.IsNullOrWhiteSpace(name)) throw new DomainException("Ürün adı boş olamaz.");
        if (price <= 0) throw new DomainException("Fiyat sıfırdan büyük olmalıdır.");
        Name = name;
        Price = price;
    }
}

// Infrastructure Layer
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).HasMaxLength(200).IsRequired();
        builder.Property(p => p.Price).HasColumnType("decimal(18,2)");
    }
}
```

---

## 5. Aggregate Root Sınırlarını Belirlememek

❌ **Yanlış Kullanım:** İç entity'lere doğrudan erişim izni vermek.

```csharp
public class Order
{
    public List<OrderItem> Items { get; set; } = new();
}

// Dışarıdan iç entity doğrudan değiştirilebilir
order.Items.Add(new OrderItem());
order.Items[0].Quantity = -5; // Tutarsız durum
order.Items.Clear(); // Aggregate kuralları bypass edilir
```

✅ **İdeal Kullanım:** Aggregate Root üzerinden tüm erişimi kontrol edin.

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(int productId, decimal price, int quantity)
    {
        if (quantity <= 0) throw new DomainException("Miktar pozitif olmalıdır.");

        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
            existing.IncreaseQuantity(quantity);
        else
            _items.Add(new OrderItem(productId, price, quantity));
    }

    public void RemoveItem(int productId)
    {
        var item = _items.FirstOrDefault(i => i.ProductId == productId)
            ?? throw new DomainException("Ürün bulunamadı.");
        _items.Remove(item);
    }
}
```