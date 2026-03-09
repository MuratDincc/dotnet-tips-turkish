# RabbitMQ

RabbitMQ, servisler arası asenkron mesajlaşma sağlar; yanlış yapılandırma mesaj kaybına ve performans sorunlarına yol açar.

---

## 1. Mesaj Kalıcılığını Sağlamamak

❌ **Yanlış Kullanım:** Mesajları kalıcı olmayan kuyrukta tutmak.

```csharp
channel.QueueDeclare("orders", durable: false, exclusive: false, autoDelete: false);
channel.BasicPublish("", "orders", null, body);
// RabbitMQ restart olduğunda tüm mesajlar kaybolur
```

✅ **İdeal Kullanım:** Durable queue ve persistent mesajlar kullanın.

```csharp
channel.QueueDeclare("orders", durable: true, exclusive: false, autoDelete: false);

var properties = channel.CreateBasicProperties();
properties.Persistent = true;

channel.BasicPublish("", "orders", properties, body);
```

---

## 2. Manuel Ack Yapmamak

❌ **Yanlış Kullanım:** AutoAck ile mesaj işlenmeden onaylamak.

```csharp
channel.BasicConsume("orders", autoAck: true, consumer); // Mesaj alınır alınmaz ack
// İşlem sırasında hata olursa mesaj kaybolur
```

✅ **İdeal Kullanım:** Manuel ack ile mesajı işlendikten sonra onaylayın.

```csharp
channel.BasicConsume("orders", autoAck: false, consumer);

consumer.Received += async (sender, args) =>
{
    try
    {
        var message = Encoding.UTF8.GetString(args.Body.ToArray());
        await ProcessOrderAsync(message);
        channel.BasicAck(args.DeliveryTag, multiple: false);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Mesaj işlenemedi");
        channel.BasicNack(args.DeliveryTag, multiple: false, requeue: true);
    }
};
```

---

## 3. MassTransit Kullanmamak

❌ **Yanlış Kullanım:** RabbitMQ client'ı doğrudan kullanarak her şeyi manuel yazmak.

```csharp
var factory = new ConnectionFactory { HostName = "localhost" };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();
channel.QueueDeclare("orders", true, false, false);
var body = Encoding.UTF8.GetBytes(JsonSerializer.Serialize(order));
channel.BasicPublish("", "orders", null, body);
// Connection yönetimi, serialization, retry, error handling hep manuel
```

✅ **İdeal Kullanım:** MassTransit ile mesajlaşma altyapısını soyutlayın.

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(context);
    });
});

// Publisher
public class OrderService
{
    private readonly IPublishEndpoint _publishEndpoint;

    public async Task CreateAsync(Order order)
    {
        await _repository.AddAsync(order);
        await _publishEndpoint.Publish(new OrderCreatedEvent { OrderId = order.Id });
    }
}

// Consumer
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var orderId = context.Message.OrderId;
        await _emailService.SendConfirmationAsync(orderId);
    }
}
```

---

## 4. Dead Letter Queue Kullanmamak

❌ **Yanlış Kullanım:** Başarısız mesajları sonsuz döngüde tekrar deneyerek kuyruğu tıkamak.

```csharp
consumer.Received += async (sender, args) =>
{
    try
    {
        await ProcessAsync(args);
        channel.BasicAck(args.DeliveryTag, false);
    }
    catch
    {
        channel.BasicNack(args.DeliveryTag, false, requeue: true); // Sürekli tekrar kuyruğa girer
    }
};
```

✅ **İdeal Kullanım:** Dead Letter Queue ile başarısız mesajları ayırın.

```csharp
// MassTransit ile otomatik DLQ
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderConsumer>(cfg =>
    {
        cfg.UseMessageRetry(r => r.Intervals(
            TimeSpan.FromSeconds(5),
            TimeSpan.FromSeconds(15),
            TimeSpan.FromSeconds(60)));
    });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.ConfigureEndpoints(context);
        // Başarısız mesajlar otomatik olarak _error kuyruğuna taşınır
    });
});
```

---

## 5. Prefetch Count Ayarlamamak

❌ **Yanlış Kullanım:** Varsayılan prefetch ile tüm mesajları tek consumer'a yüklemek.

```csharp
channel.BasicConsume("orders", false, consumer);
// Varsayılan prefetch sınırsız, tüm mesajlar tek consumer'a gider
// Diğer consumer'lar boşta kalır
```

✅ **İdeal Kullanım:** Prefetch count ile yük dağılımını dengeleyin.

```csharp
// Manuel RabbitMQ
channel.BasicQos(prefetchSize: 0, prefetchCount: 10, global: false);

// MassTransit
x.UsingRabbitMq((context, cfg) =>
{
    cfg.PrefetchCount = 16;

    cfg.ReceiveEndpoint("order-processing", e =>
    {
        e.PrefetchCount = 10;
        e.ConcurrentMessageLimit = 5;
        e.ConfigureConsumer<OrderConsumer>(context);
    });
});
```