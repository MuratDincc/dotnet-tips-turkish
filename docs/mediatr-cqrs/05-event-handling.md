# Event Handling

Event Handling, MediatR'da domain event'leri yönetmek için kullanılan bir pattern'dır. Event'ler `INotification` interface'ini implemente eder.

## Event Özellikleri

1. **Immutable**: Event'ler immutable olmalıdır
2. **Past Tense**: Event isimleri geçmiş zaman kullanmalıdır
3. **Domain Specific**: Event'ler domain'e özgü olmalıdır
4. **Idempotent**: Mümkünse idempotent olmalıdır

## Event Örnekleri

### Basit Event
```csharp
public class ProductCreatedEvent : INotification
{
    public int ProductId { get; }
    public string Name { get; }
    public decimal Price { get; }

    public ProductCreatedEvent(int productId, string name, decimal price)
    {
        ProductId = productId;
        Name = name;
        Price = price;
    }
}

public class ProductCreatedEventHandler : INotificationHandler<ProductCreatedEvent>
{
    private readonly IEmailService _emailService;
    private readonly ILogger<ProductCreatedEventHandler> _logger;

    public ProductCreatedEventHandler(
        IEmailService emailService,
        ILogger<ProductCreatedEventHandler> logger)
    {
        _emailService = emailService;
        _logger = logger;
    }

    public async Task Handle(ProductCreatedEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Product created: {ProductId}", notification.ProductId);
        
        await _emailService.SendProductCreatedEmail(
            notification.ProductId,
            notification.Name,
            notification.Price);
    }
}
```

### Complex Event
```csharp
public class OrderPlacedEvent : INotification
{
    public int OrderId { get; }
    public string CustomerEmail { get; }
    public List<OrderItem> Items { get; }
    public decimal TotalAmount { get; }

    public OrderPlacedEvent(
        int orderId,
        string customerEmail,
        List<OrderItem> items,
        decimal totalAmount)
    {
        OrderId = orderId;
        CustomerEmail = customerEmail;
        Items = items;
        TotalAmount = totalAmount;
    }
}

public class OrderPlacedEventHandler : INotificationHandler<OrderPlacedEvent>
{
    private readonly IEmailService _emailService;
    private readonly IInventoryService _inventoryService;
    private readonly ILogger<OrderPlacedEventHandler> _logger;

    public OrderPlacedEventHandler(
        IEmailService emailService,
        IInventoryService inventoryService,
        ILogger<OrderPlacedEventHandler> logger)
    {
        _emailService = emailService;
        _inventoryService = inventoryService;
        _logger = logger;
    }

    public async Task Handle(OrderPlacedEvent notification, CancellationToken cancellationToken)
    {
        _logger.LogInformation("Order placed: {OrderId}", notification.OrderId);

        // Send confirmation email
        await _emailService.SendOrderConfirmationEmail(
            notification.OrderId,
            notification.CustomerEmail,
            notification.Items,
            notification.TotalAmount);

        // Update inventory
        await _inventoryService.UpdateInventory(notification.Items);
    }
}
```

## Event Best Practices

1. **Event Publishing**
   - Event'leri handler içinde publish edin
   - Transaction içinde publish edin
   - Exception durumunda rollback yapın

2. **Event Handler Organization**
   - Her handler tek bir iş yapmalı
   - Handler'lar bağımsız olmalı
   - Cross-cutting concern'ler behavior'lara taşınmalı

3. **Error Handling**
   - Her handler kendi exception'larını yakalamalı
   - Global exception handling kullanılmalı
   - Retry mekanizması kullanılmalı

4. **Testing**
   - Handler'lar unit test edilmeli
   - Integration testler yazılmalı
   - Event flow test edilmeli

## Event Pipeline

Event'ler pipeline üzerinden geçer:

1. Validation
2. Logging
3. Handler
4. Retry

```csharp
public class EventPipelineBehavior<TNotification> : IPipelineBehavior<TNotification, Unit>
    where TNotification : INotification
{
    private readonly ILogger<EventPipelineBehavior<TNotification>> _logger;

    public EventPipelineBehavior(ILogger<EventPipelineBehavior<TNotification>> logger)
    {
        _logger = logger;
    }

    public async Task<Unit> Handle(
        TNotification notification,
        NotificationHandlerDelegate next,
        CancellationToken cancellationToken)
    {
        try
        {
            await next();
            return Unit.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling notification {NotificationName}", typeof(TNotification).Name);
            throw;
        }
    }
}
```

## Event Registration

```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    cfg.AddNotificationHandler(typeof(ProductCreatedEventHandler));
    cfg.AddNotificationHandler(typeof(OrderPlacedEventHandler));
});
``` 