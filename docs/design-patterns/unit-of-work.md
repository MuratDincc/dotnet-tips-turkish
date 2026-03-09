# Unit of Work Pattern

Unit of Work deseni, birden fazla repository işlemini tek bir transaction altında yöneterek veri tutarlılığını sağlar; yanlış kullanımlar veri bütünlüğü sorunlarına neden olur.

---

## 1. Her Repository'de Ayrı SaveChanges

❌ **Yanlış Kullanım:** Her repository kendi kaydetme işlemini yapmak.

```csharp
public class OrderRepository
{
    public async Task AddAsync(Order order)
    {
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();
    }
}

public class InventoryRepository
{
    public async Task UpdateStockAsync(int productId, int quantity)
    {
        var product = await _context.Products.FindAsync(productId);
        product.Stock -= quantity;
        await _context.SaveChangesAsync();
    }
}

// Sipariş eklenir ama stok güncellemesi başarısız olursa tutarsızlık oluşur
```

✅ **İdeal Kullanım:** Unit of Work ile tüm işlemleri tek transaction'da yönetin.

```csharp
public interface IUnitOfWork : IDisposable
{
    IOrderRepository Orders { get; }
    IInventoryRepository Inventory { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Orders = new OrderRepository(context);
        Inventory = new InventoryRepository(context);
    }

    public IOrderRepository Orders { get; }
    public IInventoryRepository Inventory { get; }

    public Task<int> SaveChangesAsync(CancellationToken ct) => _context.SaveChangesAsync(ct);
    public void Dispose() => _context.Dispose();
}
```

---

## 2. DbContext'in Zaten Unit of Work Olduğunu Görmezden Gelmek

❌ **Yanlış Kullanım:** EF Core üzerine gereksiz Unit of Work soyutlaması yazmak.

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;
    private IOrderRepository _orders;
    private IProductRepository _products;
    private ICustomerRepository _customers;
    // Her yeni entity için property eklenmeli

    public IOrderRepository Orders => _orders ??= new OrderRepository(_context);
    public IProductRepository Products => _products ??= new ProductRepository(_context);
    // ...
}
```

✅ **İdeal Kullanım:** Basit senaryolarda DbContext'i doğrudan Unit of Work olarak kullanın.

```csharp
public class OrderService
{
    private readonly AppDbContext _context;

    public OrderService(AppDbContext context) => _context = context;

    public async Task PlaceOrderAsync(Order order)
    {
        _context.Orders.Add(order);

        var product = await _context.Products.FindAsync(order.ProductId);
        product.Stock -= order.Quantity;

        await _context.SaveChangesAsync(); // Tek transaction, EF Core bunu zaten yönetir
    }
}
```

---

## 3. Transaction Scope'unu Yanlış Yönetmek

❌ **Yanlış Kullanım:** Transaction scope'unu çok geniş tutmak.

```csharp
public async Task ProcessAsync()
{
    using var transaction = await _context.Database.BeginTransactionAsync();

    var orders = await _context.Orders.ToListAsync();
    foreach (var order in orders)
    {
        await _emailService.SendAsync(order.Email); // Harici servis çağrısı transaction içinde
        order.IsNotified = true;
    }

    await _context.SaveChangesAsync();
    await transaction.CommitAsync(); // E-posta servisi yavaşlarsa transaction uzar
}
```

✅ **İdeal Kullanım:** Transaction scope'unu minimum tutun, harici çağrıları dışarıda bırakın.

```csharp
public async Task ProcessAsync()
{
    var orders = await _context.Orders
        .Where(o => !o.IsNotified)
        .ToListAsync();

    foreach (var order in orders)
    {
        order.IsNotified = true;
    }
    await _context.SaveChangesAsync(); // Sadece DB işlemi transaction içinde

    foreach (var order in orders)
    {
        await _emailService.SendAsync(order.Email); // Transaction dışında
    }
}
```

---

## 4. Concurrent SaveChanges Çağrıları

❌ **Yanlış Kullanım:** Aynı DbContext üzerinde paralel SaveChanges çağrısı yapmak.

```csharp
public async Task UpdateAllAsync(List<Order> orders)
{
    var tasks = orders.Select(async order =>
    {
        order.Status = OrderStatus.Processed;
        await _context.SaveChangesAsync(); // DbContext thread-safe değil!
    });

    await Task.WhenAll(tasks);
}
```

✅ **İdeal Kullanım:** Tüm değişiklikleri toplu yapıp tek SaveChanges çağırın.

```csharp
public async Task UpdateAllAsync(List<Order> orders)
{
    foreach (var order in orders)
    {
        order.Status = OrderStatus.Processed;
    }

    await _context.SaveChangesAsync(); // Tek çağrı, tüm değişiklikler
}
```

---

## 5. Unit of Work ile Concurrency Kontrolü

❌ **Yanlış Kullanım:** Concurrency conflict'leri görmezden gelmek.

```csharp
public async Task UpdatePriceAsync(int productId, decimal newPrice)
{
    var product = await _context.Products.FindAsync(productId);
    product.Price = newPrice;
    await _context.SaveChangesAsync(); // Başka biri aynı anda güncellerse veri kaybı
}
```

✅ **İdeal Kullanım:** Optimistic concurrency ile çakışmaları yönetin.

```csharp
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}

public async Task UpdatePriceAsync(int productId, decimal newPrice)
{
    var product = await _context.Products.FindAsync(productId);
    product.Price = newPrice;

    try
    {
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException)
    {
        // Çakışma durumunda yeniden oku ve kullanıcıyı bilgilendir
        throw new ConflictException("Ürün başka bir kullanıcı tarafından güncellenmiş.");
    }
}
```