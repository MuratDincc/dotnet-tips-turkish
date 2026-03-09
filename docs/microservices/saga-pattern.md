# Saga Pattern

Saga Pattern, microservice'ler arası dağıtık işlemlerde veri tutarlılığını sağlar; yanlış uygulamalar tutarsız veriye ve kurtarılamaz hatalara yol açar.

---

## 1. Dağıtık Transaction Kullanmak

❌ **Yanlış Kullanım:** Microservice'ler arası two-phase commit ile transaction yönetmek.

```csharp
public async Task CreateOrderAsync(OrderRequest request)
{
    using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

    await _orderDb.CreateOrderAsync(request);     // Order DB
    await _paymentDb.ChargeAsync(request.Total);   // Payment DB - farklı servis!
    await _inventoryDb.ReserveAsync(request.Items); // Inventory DB - farklı servis!

    scope.Complete(); // Dağıtık transaction - tight coupling, performans sorunu
}
```

✅ **İdeal Kullanım:** Saga pattern ile eventual consistency sağlayın.

```csharp
public class CreateOrderSaga
{
    public async Task ExecuteAsync(OrderRequest request)
    {
        // Step 1: Sipariş oluştur
        var orderId = await _orderService.CreateAsync(request);

        try
        {
            // Step 2: Ödeme al
            await _paymentService.ChargeAsync(orderId, request.Total);

            // Step 3: Stok ayır
            await _inventoryService.ReserveAsync(orderId, request.Items);
        }
        catch (PaymentFailedException)
        {
            await _orderService.CancelAsync(orderId);
            throw;
        }
        catch (InsufficientStockException)
        {
            await _paymentService.RefundAsync(orderId);
            await _orderService.CancelAsync(orderId);
            throw;
        }
    }
}
```

---

## 2. Compensation Mantığını Yazmamak

❌ **Yanlış Kullanım:** Hata durumunda geri alma işlemi yapmamak.

```csharp
public async Task ProcessOrderAsync(Order order)
{
    await _paymentService.ChargeAsync(order);
    await _shippingService.CreateShipmentAsync(order); // Başarısız olursa ödeme geri alınmaz
    await _notificationService.SendConfirmationAsync(order);
}
```

✅ **İdeal Kullanım:** Her adım için compensation aksiyonu tanımlayın.

```csharp
public class OrderSagaStep
{
    public Func<Task> Execute { get; init; }
    public Func<Task> Compensate { get; init; }
}

public class OrderSagaOrchestrator
{
    public async Task ExecuteAsync(Order order)
    {
        var steps = new List<OrderSagaStep>
        {
            new()
            {
                Execute = () => _paymentService.ChargeAsync(order),
                Compensate = () => _paymentService.RefundAsync(order.Id)
            },
            new()
            {
                Execute = () => _inventoryService.ReserveAsync(order),
                Compensate = () => _inventoryService.ReleaseAsync(order.Id)
            },
            new()
            {
                Execute = () => _shippingService.CreateAsync(order),
                Compensate = () => _shippingService.CancelAsync(order.Id)
            }
        };

        var completedSteps = new Stack<OrderSagaStep>();

        foreach (var step in steps)
        {
            try
            {
                await step.Execute();
                completedSteps.Push(step);
            }
            catch
            {
                while (completedSteps.Count > 0)
                {
                    var compensate = completedSteps.Pop();
                    await compensate.Compensate();
                }
                throw;
            }
        }
    }
}
```

---

## 3. Choreography ve Orchestration Karışımı

❌ **Yanlış Kullanım:** Event-driven ve command-driven yaklaşımları karıştırmak.

```csharp
public class OrderService
{
    public async Task CreateAsync(Order order)
    {
        await _repository.AddAsync(order);
        await _messageBus.PublishAsync(new OrderCreated(order.Id)); // Event (choreography)
        await _paymentService.ChargeAsync(order.Total);             // Direct call (orchestration)
    }
}
```

✅ **İdeal Kullanım:** Tutarlı bir yaklaşım seçin. Orchestration örneği:

```csharp
public class OrderSagaOrchestrator : IRequestHandler<CreateOrderCommand, OrderResult>
{
    private readonly ISagaStateRepository _sagaRepo;
    private readonly IMessageBus _bus;

    public async Task<OrderResult> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var saga = new OrderSagaState(request.OrderId);
        await _sagaRepo.SaveAsync(saga);

        await _bus.SendAsync(new ChargePayment(saga.Id, request.OrderId, request.Total));
        return new OrderResult(request.OrderId, "Processing");
    }

    public async Task HandlePaymentCharged(PaymentCharged @event)
    {
        var saga = await _sagaRepo.GetAsync(@event.SagaId);
        saga.PaymentCompleted = true;
        await _bus.SendAsync(new ReserveInventory(saga.Id, saga.OrderId));
    }

    public async Task HandlePaymentFailed(PaymentFailed @event)
    {
        var saga = await _sagaRepo.GetAsync(@event.SagaId);
        saga.Status = SagaStatus.Failed;
        await _bus.SendAsync(new CancelOrder(saga.OrderId));
    }
}
```

---

## 4. Saga State Yönetimi Yapmamak

❌ **Yanlış Kullanım:** Saga durumunu izlememek.

```csharp
public async Task ProcessAsync(Order order)
{
    await _paymentService.ChargeAsync(order);
    // Uygulama çökerse saga'nın hangi adımda olduğu bilinmez
    await _inventoryService.ReserveAsync(order);
}
```

✅ **İdeal Kullanım:** Saga state'ini persist ederek izlenebilirlik sağlayın.

```csharp
public class OrderSagaState
{
    public Guid Id { get; set; }
    public int OrderId { get; set; }
    public SagaStatus Status { get; set; }
    public bool PaymentCompleted { get; set; }
    public bool InventoryReserved { get; set; }
    public bool ShipmentCreated { get; set; }
    public DateTime StartedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public string FailureReason { get; set; }

    public bool IsComplete => PaymentCompleted && InventoryReserved && ShipmentCreated;
}

// Saga recovery worker
public class SagaRecoveryService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var stuckSagas = await _sagaRepo.GetStuckAsync(TimeSpan.FromMinutes(5));
            foreach (var saga in stuckSagas)
            {
                await ResumeOrCompensateAsync(saga);
            }
            await Task.Delay(TimeSpan.FromMinutes(1), ct);
        }
    }
}
```

---

## 5. Timeout ve Deadletter Yönetimi

❌ **Yanlış Kullanım:** Saga adımlarında timeout belirlememek.

```csharp
public async Task HandleOrderCreated(OrderCreated @event)
{
    await _paymentService.ChargeAsync(@event.OrderId);
    // Payment servisi yanıt vermezse saga sonsuza kadar bekler
}
```

✅ **İdeal Kullanım:** Timeout ve deadletter queue ile takılı saga'ları yönetin.

```csharp
public class SagaTimeoutManager : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var timedOutSagas = await _sagaRepo.GetTimedOutAsync(TimeSpan.FromMinutes(2));

            foreach (var saga in timedOutSagas)
            {
                _logger.LogWarning("Saga timeout: {SagaId}, adım: {Step}", saga.Id, saga.CurrentStep);
                saga.Status = SagaStatus.TimedOut;

                await CompensateAsync(saga);
                await _deadLetterQueue.PublishAsync(new SagaTimedOut(saga.Id, saga.CurrentStep));
            }

            await Task.Delay(TimeSpan.FromSeconds(30), ct);
        }
    }
}
```