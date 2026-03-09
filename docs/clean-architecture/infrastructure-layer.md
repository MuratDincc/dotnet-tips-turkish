# Infrastructure Layer

Infrastructure katmanı, veritabanı, dosya sistemi ve harici servisler gibi altyapı detaylarını uygular; yanlış kullanımlar katmanlar arası bağımlılığı artırır.

---

## 1. Bağımlılık Yönünü Ters Çevirmek

❌ **Yanlış Kullanım:** Application katmanının Infrastructure'a referans vermesi.

```csharp
// Application Layer projesi -> Infrastructure projesini referans ediyor
using MyApp.Infrastructure.Data;
using MyApp.Infrastructure.Services;

public class OrderService
{
    private readonly AppDbContext _context; // Concrete Infrastructure sınıfı
    private readonly SmtpEmailService _emailService;
}
```

✅ **İdeal Kullanım:** Application katmanında interface tanımlayın, Infrastructure uygulasın.

```csharp
// Application Layer - interface tanımı
public interface IOrderRepository
{
    Task<Order> GetByIdAsync(int id);
    Task AddAsync(Order order);
}

// Infrastructure Layer - implementasyon
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order> GetByIdAsync(int id)
        => await _context.Orders.FindAsync(id);

    public async Task AddAsync(Order order)
        => await _context.Orders.AddAsync(order);
}

// Program.cs - DI kaydı
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

---

## 2. DbContext Konfigürasyonunu Dağıtmak

❌ **Yanlış Kullanım:** Entity konfigürasyonlarını OnModelCreating içinde toplamak.

```csharp
public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>(e =>
        {
            e.HasKey(o => o.Id);
            e.Property(o => o.TotalPrice).HasColumnType("decimal(18,2)");
            // 50+ satır konfigürasyon
        });

        modelBuilder.Entity<Product>(e =>
        {
            // 50+ satır daha
        });
        // Dosya yüzlerce satır olur
    }
}
```

✅ **İdeal Kullanım:** Her entity için ayrı konfigürasyon dosyası oluşturun.

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.TotalPrice).HasColumnType("decimal(18,2)");
        builder.HasMany(o => o.Items).WithOne().HasForeignKey(i => i.OrderId);
    }
}

public class AppDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

---

## 3. Harici Servis Çağrılarını Soyutlamamak

❌ **Yanlış Kullanım:** Harici API çağrılarını doğrudan servis içinde yapmak.

```csharp
public class PaymentService
{
    public async Task<bool> ChargeAsync(decimal amount, string cardToken)
    {
        using var client = new HttpClient();
        var response = await client.PostAsJsonAsync("https://api.stripe.com/charges", new
        {
            amount = amount * 100,
            currency = "try",
            source = cardToken
        });
        return response.IsSuccessStatusCode;
    }
}
```

✅ **İdeal Kullanım:** Harici servisi interface ile soyutlayın ve HttpClientFactory kullanın.

```csharp
// Application Layer
public interface IPaymentGateway
{
    Task<PaymentResult> ChargeAsync(decimal amount, string cardToken);
}

// Infrastructure Layer
public class StripePaymentGateway : IPaymentGateway
{
    private readonly HttpClient _client;

    public StripePaymentGateway(HttpClient client) => _client = client;

    public async Task<PaymentResult> ChargeAsync(decimal amount, string cardToken)
    {
        var response = await _client.PostAsJsonAsync("/charges", new
        {
            amount = amount * 100,
            currency = "try",
            source = cardToken
        });

        return response.IsSuccessStatusCode
            ? PaymentResult.Success()
            : PaymentResult.Failure("Ödeme başarısız");
    }
}

// DI kaydı
builder.Services.AddHttpClient<IPaymentGateway, StripePaymentGateway>(client =>
{
    client.BaseAddress = new Uri("https://api.stripe.com/");
});
```

---

## 4. Repository'de Gereksiz Soyutlama

❌ **Yanlış Kullanım:** Basit CRUD için karmaşık generic repository hiyerarşisi kurmak.

```csharp
public interface IRepository<T> { }
public interface IReadRepository<T> : IRepository<T> { }
public interface IWriteRepository<T> : IRepository<T> { }
public abstract class BaseRepository<T> : IReadRepository<T>, IWriteRepository<T> { }
public class OrderRepository : BaseRepository<Order> { }
// 4 katman soyutlama, sadece CRUD için
```

✅ **İdeal Kullanım:** İhtiyaca göre basit ve yeterli soyutlama yapın.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<IEnumerable<Order>> GetByCustomerAsync(int customerId);
    Task AddAsync(Order order);
}

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(int id)
        => await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id);

    public async Task<IEnumerable<Order>> GetByCustomerAsync(int customerId)
        => await _context.Orders
            .Where(o => o.CustomerId == customerId)
            .ToListAsync();

    public async Task AddAsync(Order order)
        => await _context.Orders.AddAsync(order);
}
```

---

## 5. Infrastructure Servis Kayıtlarını Dağıtmak

❌ **Yanlış Kullanım:** Tüm DI kayıtlarını Program.cs'e yığmak.

```csharp
// Program.cs - 100+ satır servis kaydı
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<ICustomerRepository, CustomerRepository>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
builder.Services.AddScoped<IEmailService, SmtpEmailService>();
// ...50 satır daha
```

✅ **İdeal Kullanım:** Katman bazlı extension method ile kayıtları organize edin.

```csharp
// Infrastructure Layer
public static class InfrastructureServiceRegistration
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IPaymentGateway, StripePaymentGateway>();

        return services;
    }
}

// Program.cs - temiz ve okunabilir
builder.Services.AddInfrastructureServices(builder.Configuration);
builder.Services.AddApplicationServices();
```