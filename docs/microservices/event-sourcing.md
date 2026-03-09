# Event Sourcing

Event Sourcing, uygulamanın durumunu olaylar üzerinden saklayarak tam bir değişiklik geçmişi sunar; yanlış uygulamalar karmaşık sorgulara ve performans sorunlarına yol açar.

---

## 1. Mutable State ile Event Sourcing Karışımı

❌ **Yanlış Kullanım:** Hem event store hem de doğrudan state güncellemesi yapmak.

```csharp
public class BankAccount
{
    public decimal Balance { get; set; }

    public void Deposit(decimal amount)
    {
        Balance += amount; // Doğrudan state değişikliği
        _events.Add(new MoneyDeposited(amount)); // Event de ekleniyor
        _context.SaveChanges(); // State kaydediliyor
    }
}
```

✅ **İdeal Kullanım:** State'i yalnızca event'lerden türetin.

```csharp
public class BankAccount
{
    private readonly List<IDomainEvent> _uncommittedEvents = new();
    public decimal Balance { get; private set; }

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new DomainException("Tutar pozitif olmalıdır.");
        Apply(new MoneyDeposited(Id, amount, DateTime.UtcNow));
    }

    private void Apply(IDomainEvent @event)
    {
        When(@event);
        _uncommittedEvents.Add(@event);
    }

    private void When(IDomainEvent @event)
    {
        switch (@event)
        {
            case MoneyDeposited e: Balance += e.Amount; break;
            case MoneyWithdrawn e: Balance -= e.Amount; break;
        }
    }
}
```

---

## 2. Event Versiyonlama Yapmamak

❌ **Yanlış Kullanım:** Event yapısını değiştirip eski event'leri kırmak.

```csharp
// v1
public record OrderCreated(int OrderId, int CustomerId, decimal Total);

// v2 - breaking change, eski event'ler deserialize edilemez
public record OrderCreated(int OrderId, int CustomerId, decimal Total, string Currency, DateTime CreatedAt);
```

✅ **İdeal Kullanım:** Event versiyonlama ve upcasting ile geriye uyumluluk sağlayın.

```csharp
public record OrderCreatedV1(int OrderId, int CustomerId, decimal Total);

public record OrderCreatedV2(int OrderId, int CustomerId, decimal Total, string Currency, DateTime CreatedAt);

public class OrderCreatedUpcaster : IEventUpcaster
{
    public IDomainEvent Upcast(IDomainEvent @event)
    {
        if (@event is OrderCreatedV1 v1)
        {
            return new OrderCreatedV2(v1.OrderId, v1.CustomerId, v1.Total, "TRY", DateTime.UtcNow);
        }
        return @event;
    }
}
```

---

## 3. Snapshot Kullanmamak

❌ **Yanlış Kullanım:** Her okumada tüm event geçmişini tekrar oynatmak.

```csharp
public async Task<BankAccount> GetAccountAsync(Guid accountId)
{
    var events = await _eventStore.GetEventsAsync(accountId); // 10.000+ event olabilir
    var account = new BankAccount();
    foreach (var @event in events)
    {
        account.Apply(@event); // Her okumada tüm geçmiş tekrar oynatılır
    }
    return account;
}
```

✅ **İdeal Kullanım:** Belirli aralıklarla snapshot alarak performansı artırın.

```csharp
public async Task<BankAccount> GetAccountAsync(Guid accountId)
{
    var snapshot = await _snapshotStore.GetLatestAsync<BankAccount>(accountId);
    var fromVersion = snapshot?.Version ?? 0;

    var events = await _eventStore.GetEventsAsync(accountId, fromVersion);

    var account = snapshot ?? new BankAccount();
    foreach (var @event in events)
    {
        account.Apply(@event);
    }

    if (events.Count > 100) // Her 100 event'te snapshot al
    {
        await _snapshotStore.SaveAsync(accountId, account, account.Version);
    }

    return account;
}
```

---

## 4. Projection'ları Senkron Güncellemek

❌ **Yanlış Kullanım:** Event kaydederken projection'ı aynı transaction'da güncellemek.

```csharp
public async Task HandleAsync(OrderCreated @event)
{
    await _eventStore.AppendAsync(@event);

    // Aynı transaction'da - event store ve read model sıkı bağlı
    var summary = await _readDb.OrderSummaries.FindAsync(@event.OrderId);
    summary.Status = "Created";
    summary.Total = @event.Total;
    await _readDb.SaveChangesAsync();
}
```

✅ **İdeal Kullanım:** Projection'ları asenkron olarak güncelleyin.

```csharp
// Event kaydı
public async Task HandleAsync(OrderCreated @event)
{
    await _eventStore.AppendAsync(@event);
}

// Ayrı projection worker
public class OrderSummaryProjection : IEventHandler<OrderCreated>
{
    private readonly ReadDbContext _readDb;

    public async Task HandleAsync(OrderCreated @event, CancellationToken ct)
    {
        var summary = new OrderSummary
        {
            OrderId = @event.OrderId,
            CustomerId = @event.CustomerId,
            Total = @event.Total,
            Status = "Created"
        };

        await _readDb.OrderSummaries.AddAsync(summary, ct);
        await _readDb.SaveChangesAsync(ct);
    }
}
```

---

## 5. Idempotency Sağlamamak

❌ **Yanlış Kullanım:** Aynı event'in tekrar işlenmesine karşı koruma yapmamak.

```csharp
public class InventoryProjection : IEventHandler<OrderCreated>
{
    public async Task HandleAsync(OrderCreated @event, CancellationToken ct)
    {
        var product = await _context.Products.FindAsync(@event.ProductId);
        product.Stock -= @event.Quantity; // Event tekrar işlenirse stok yanlış azalır
        await _context.SaveChangesAsync(ct);
    }
}
```

✅ **İdeal Kullanım:** Event position tracking ile idempotency sağlayın.

```csharp
public class InventoryProjection : IEventHandler<OrderCreated>
{
    public async Task HandleAsync(OrderCreated @event, CancellationToken ct)
    {
        var processed = await _context.ProcessedEvents
            .AnyAsync(e => e.EventId == @event.EventId, ct);

        if (processed) return; // Zaten işlenmiş

        var product = await _context.Products.FindAsync(@event.ProductId);
        product.Stock -= @event.Quantity;

        _context.ProcessedEvents.Add(new ProcessedEvent { EventId = @event.EventId });
        await _context.SaveChangesAsync(ct);
    }
}
```