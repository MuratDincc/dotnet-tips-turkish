# Repository Pattern

Repository deseni, veri erişim mantığını soyutlayarak iş mantığını veritabanı detaylarından ayırır; yanlış uygulamalar ise gereksiz soyutlama katmanlarına ve performans sorunlarına yol açar.

---

## 1. Generic Repository Anti-Pattern

❌ **Yanlış Kullanım:** Her entity için aynı CRUD metodlarını sunan generic repository.

```csharp
public interface IRepository<T>
{
    Task<T> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
}

// Her entity aynı interface'i kullanır, özel sorgular yapılamaz
var orders = await _orderRepository.GetAllAsync(); // Tüm siparişleri çeker!
```

✅ **İdeal Kullanım:** Entity'ye özel repository interface'leri tanımlayın.

```csharp
public interface IOrderRepository
{
    Task<Order> GetByIdAsync(int id);
    Task<IEnumerable<Order>> GetPendingOrdersAsync();
    Task<IEnumerable<Order>> GetByCustomerAsync(int customerId);
    Task AddAsync(Order order);
}

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<IEnumerable<Order>> GetPendingOrdersAsync()
    {
        return await _context.Orders
            .Where(o => o.Status == OrderStatus.Pending)
            .ToListAsync();
    }
}
```

---

## 2. DbContext'i Sızdırmak

❌ **Yanlış Kullanım:** Repository'den `IQueryable` döndürerek DbContext'i dışarı sızdırmak.

```csharp
public interface IProductRepository
{
    IQueryable<Product> GetAll(); // Çağıran taraf istedigi sorguyu yazabilir
}

// Controller'da
var products = await _repo.GetAll()
    .Include(p => p.Category)
    .Where(p => p.Price > 100)
    .ToListAsync(); // Repository soyutlaması anlamsızlaşıyor
```

✅ **İdeal Kullanım:** Repository'den somut sonuçlar döndürün.

```csharp
public interface IProductRepository
{
    Task<IEnumerable<Product>> GetExpensiveProductsAsync(decimal minPrice);
    Task<Product?> GetWithCategoryAsync(int id);
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context) => _context = context;

    public async Task<Product?> GetWithCategoryAsync(int id)
    {
        return await _context.Products
            .Include(p => p.Category)
            .FirstOrDefaultAsync(p => p.Id == id);
    }
}
```

---

## 3. Repository İçinde İş Mantığı

❌ **Yanlış Kullanım:** İş mantığını repository katmanına koymak.

```csharp
public class OrderRepository
{
    public async Task CreateOrderAsync(Order order)
    {
        if (order.Items.Count == 0)
            throw new BusinessException("Sipariş boş olamaz");

        order.TotalPrice = order.Items.Sum(i => i.Price * i.Quantity);
        order.Status = OrderStatus.Pending;

        await _context.Orders.AddAsync(order);
        await _context.SaveChangesAsync();
    }
}
```

✅ **İdeal Kullanım:** İş mantığını servis katmanında tutun, repository sadece veri erişimi yapsın.

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public async Task CreateOrderAsync(Order order)
    {
        if (order.Items.Count == 0)
            throw new BusinessException("Sipariş boş olamaz");

        order.CalculateTotal();
        order.SetStatus(OrderStatus.Pending);

        await _repository.AddAsync(order);
    }
}

public class OrderRepository : IOrderRepository
{
    public async Task AddAsync(Order order)
    {
        await _context.Orders.AddAsync(order);
        await _context.SaveChangesAsync();
    }
}
```

---

## 4. Her Repository İçin Ayrı SaveChanges

❌ **Yanlış Kullanım:** Her repository kendi `SaveChangesAsync` çağrısını yapmak.

```csharp
public async Task PlaceOrderAsync(Order order, Payment payment)
{
    await _orderRepository.AddAsync(order);     // SaveChangesAsync çağrılır
    await _paymentRepository.AddAsync(payment);  // Ayrı SaveChangesAsync çağrılır
    // İlk başarılı ikinci başarısız olursa tutarsızlık oluşur
}
```

✅ **İdeal Kullanım:** Unit of Work ile tek bir transaction kullanın.

```csharp
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    IPaymentRepository Payments { get; }
    Task<int> SaveChangesAsync();
}

public async Task PlaceOrderAsync(Order order, Payment payment)
{
    _unitOfWork.Orders.Add(order);
    _unitOfWork.Payments.Add(payment);
    await _unitOfWork.SaveChangesAsync(); // Tek transaction
}
```

---

## 5. Specification Pattern Kullanmamak

❌ **Yanlış Kullanım:** Filtreleme kombinasyonları için çok sayıda repository metodu yazmak.

```csharp
public interface IProductRepository
{
    Task<IEnumerable<Product>> GetByCategoryAsync(int categoryId);
    Task<IEnumerable<Product>> GetByPriceRangeAsync(decimal min, decimal max);
    Task<IEnumerable<Product>> GetByCategoryAndPriceAsync(int categoryId, decimal min, decimal max);
    Task<IEnumerable<Product>> GetActiveByCategoryAsync(int categoryId);
    // Kombinasyonlar çoğaldıkça metod sayısı patlar
}
```

✅ **İdeal Kullanım:** Specification Pattern ile esnek filtreleme yapın.

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
}

public class ActiveProductSpec : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.IsActive;
}

public class PriceRangeSpec : Specification<Product>
{
    private readonly decimal _min, _max;
    public PriceRangeSpec(decimal min, decimal max) { _min = min; _max = max; }

    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.Price >= _min && p.Price <= _max;
}

public interface IProductRepository
{
    Task<IEnumerable<Product>> GetAsync(Specification<Product> spec);
}
```