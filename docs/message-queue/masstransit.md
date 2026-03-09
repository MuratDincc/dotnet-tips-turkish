# MassTransit

MassTransit, .NET için mesaj tabanlı uygulamaları kolaylaştıran bir framework'tür; yanlış kullanımlar mesaj kaybına ve consumer sorunlarına yol açar.

---

## 1. Consumer'da Exception Yönetimi

❌ **Yanlış Kullanım:** Exception'ı sessizce yutmak.

```csharp
public class OrderConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        try
        {
            await ProcessOrderAsync(context.Message);
        }
        catch (Exception)
        {
            // Hata yutuldu, mesaj ack'lanır, veri kaybolur
        }
    }
}
```

✅ **İdeal Kullanım:** Retry policy tanımlayıp hataları MassTransit'e bırakın.

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderConsumer>(cfg =>
    {
        cfg.UseMessageRetry(r => r.Incremental(3,
            TimeSpan.FromSeconds(1),
            TimeSpan.FromSeconds(5)));
    });
});

public class OrderConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        await ProcessOrderAsync(context.Message);
        // Exception fırlarsa MassTransit retry yapar, sonra _error kuyruğuna taşır
    }
}
```

---

## 2. Publish ve Send Farkını Bilmemek

❌ **Yanlış Kullanım:** Command için Publish, event için Send kullanmak.

```csharp
// Event'i Send ile göndermek - sadece bir consumer alır
await sendEndpoint.Send(new OrderCreatedEvent { OrderId = 1 });

// Command'ı Publish ile göndermek - birden fazla consumer alır
await publishEndpoint.Publish(new ProcessPaymentCommand { OrderId = 1 });
```

✅ **İdeal Kullanım:** Event'ler için Publish, command'lar için Send kullanın.

```csharp
// Event: Birden fazla consumer dinleyebilir (fan-out)
await _publishEndpoint.Publish(new OrderCreatedEvent
{
    OrderId = order.Id,
    CustomerId = order.CustomerId
});

// Command: Tek bir consumer'a yönlendirilir (point-to-point)
var endpoint = await _sendEndpointProvider.GetSendEndpoint(
    new Uri("queue:payment-processing"));
await endpoint.Send(new ProcessPaymentCommand
{
    OrderId = order.Id,
    Amount = order.TotalPrice
});
```

---

## 3. Message Contract'ı Yanlış Tasarlamak

❌ **Yanlış Kullanım:** Büyük nesneleri mesaj olarak göndermek.

```csharp
public class OrderCreatedEvent
{
    public Order Order { get; set; }         // Tüm entity grafiği
    public Customer Customer { get; set; }    // İlişkili entity'ler
    public List<Product> Products { get; set; } // Büyük liste
}
```

✅ **İdeal Kullanım:** Mesajları küçük ve self-contained tutun.

```csharp
public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public int CustomerId { get; init; }
    public decimal TotalPrice { get; init; }
    public DateTime CreatedAt { get; init; }
}

// Consumer detaylı veriyi kendi veri kaynağından çeker
public class OrderNotificationConsumer : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var orderId = context.Message.OrderId;
        var orderDetails = await _orderService.GetDetailsAsync(orderId);
        await _emailService.SendOrderConfirmationAsync(orderDetails);
    }
}
```

---

## 4. Saga State Machine Kullanmamak

❌ **Yanlış Kullanım:** Dağıtık iş akışını manuel yönetmek.

```csharp
public class OrderConsumer : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        await _paymentService.ChargeAsync(context.Message.OrderId);
        await _inventoryService.ReserveAsync(context.Message.OrderId);
        await _shippingService.CreateAsync(context.Message.OrderId);
        // Bir adım başarısız olursa tüm flow bozulur
    }
}
```

✅ **İdeal Kullanım:** MassTransit Saga State Machine ile yönetin.

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderCreated, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentCompleted, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(c => c.Message.OrderId));

        Initially(
            When(OrderCreated)
                .Then(context => context.Saga.OrderId = context.Message.OrderId)
                .Publish(context => new ProcessPayment { OrderId = context.Saga.OrderId })
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentCompleted)
                .Publish(context => new ReserveInventory { OrderId = context.Saga.OrderId })
                .TransitionTo(AwaitingInventory),
            When(PaymentFailed)
                .Publish(context => new CancelOrder { OrderId = context.Saga.OrderId })
                .TransitionTo(Cancelled));
    }
}
```

---

## 5. Outbox Pattern Kullanmamak

❌ **Yanlış Kullanım:** DB kayıt ve mesaj gönderimini ayrı işlemde yapmak.

```csharp
public async Task CreateOrderAsync(Order order)
{
    await _context.Orders.AddAsync(order);
    await _context.SaveChangesAsync();

    await _publishEndpoint.Publish(new OrderCreatedEvent { OrderId = order.Id });
    // SaveChanges başarılı ama Publish başarısız olabilir - tutarsızlık
}
```

✅ **İdeal Kullanım:** MassTransit Outbox ile atomik mesaj gönderimi sağlayın.

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.UseSqlServer();
        o.UseBusOutbox();
    });
});

public class OrderService
{
    public async Task CreateOrderAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
        await _publishEndpoint.Publish(new OrderCreatedEvent { OrderId = order.Id });
        await _context.SaveChangesAsync();
        // Mesaj ve DB kaydı aynı transaction'da, Outbox otomatik yönetir
    }
}
```