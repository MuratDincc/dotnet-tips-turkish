# Dependency Inversion Principle (DIP)

Üst seviye modüller alt seviye modüllere değil, soyutlamalara bağımlı olmalıdır; aksi halde değişikliklere kırılgan ve test edilemez kod ortaya çıkar.

---

## 1. Concrete Sınıfa Doğrudan Bağımlılık

❌ **Yanlış Kullanım:** Serviste concrete sınıfı doğrudan kullanmak.

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repository = new SqlOrderRepository();
    private readonly SmtpEmailSender _emailSender = new SmtpEmailSender();

    public async Task PlaceOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _emailSender.SendAsync(order.CustomerEmail, "Sipariş alındı");
    }
}
```

✅ **İdeal Kullanım:** Interface üzerinden bağımlılık alın.

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly IEmailSender _emailSender;

    public OrderService(IOrderRepository repository, IEmailSender emailSender)
    {
        _repository = repository;
        _emailSender = emailSender;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        await _repository.SaveAsync(order);
        await _emailSender.SendAsync(order.CustomerEmail, "Sipariş alındı");
    }
}
```

---

## 2. `new` Anahtar Kelimesi ile Sıkı Bağlılık

❌ **Yanlış Kullanım:** Bağımlılıkları metod içinde `new` ile oluşturmak.

```csharp
public class ReportService
{
    public async Task<byte[]> GenerateAsync()
    {
        var repository = new ReportRepository(new SqlConnection("..."));
        var formatter = new PdfFormatter();
        var data = await repository.GetDataAsync();
        return formatter.Format(data);
    }
}
```

✅ **İdeal Kullanım:** Bağımlılıkları constructor injection ile alın.

```csharp
public class ReportService
{
    private readonly IReportRepository _repository;
    private readonly IReportFormatter _formatter;

    public ReportService(IReportRepository repository, IReportFormatter formatter)
    {
        _repository = repository;
        _formatter = formatter;
    }

    public async Task<byte[]> GenerateAsync()
    {
        var data = await _repository.GetDataAsync();
        return _formatter.Format(data);
    }
}
```

---

## 3. Altyapı Detaylarının Domain'e Sızması

❌ **Yanlış Kullanım:** Domain katmanında altyapı kütüphanelerine referans vermek.

```csharp
// Domain katmanında
using Microsoft.EntityFrameworkCore;
using StackExchange.Redis;

public class ProductService
{
    private readonly DbContext _context;
    private readonly IConnectionMultiplexer _redis;

    public async Task<Product> GetAsync(int id)
    {
        var cached = await _redis.GetDatabase().StringGetAsync($"product:{id}");
        if (cached.HasValue) return JsonSerializer.Deserialize<Product>(cached);

        return await _context.Set<Product>().FindAsync(id);
    }
}
```

✅ **İdeal Kullanım:** Domain katmanını altyapıdan soyutlayın.

```csharp
// Domain katmanında - altyapı referansı yok
public interface IProductRepository
{
    Task<Product> GetByIdAsync(int id);
}

// Infrastructure katmanında
public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly IDistributedCache _cache;

    public CachedProductRepository(IProductRepository inner, IDistributedCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Product> GetByIdAsync(int id)
    {
        var cached = await _cache.GetStringAsync($"product:{id}");
        if (cached != null) return JsonSerializer.Deserialize<Product>(cached);

        return await _inner.GetByIdAsync(id);
    }
}
```

---

## 4. Static Yardımcı Sınıf Bağımlılığı

❌ **Yanlış Kullanım:** Static helper sınıfları kullanarak test edilemez kod yazmak.

```csharp
public class UserService
{
    public async Task RegisterAsync(User user)
    {
        user.PasswordHash = PasswordHelper.Hash(user.Password);
        await DbHelper.SaveAsync(user);
        EmailHelper.Send(user.Email, "Hoşgeldiniz");
    }
}
```

✅ **İdeal Kullanım:** Helper'ları interface ile sarmalayarak inject edilebilir hale getirin.

```csharp
public interface IPasswordHasher
{
    string Hash(string password);
}

public class UserService
{
    private readonly IUserRepository _repository;
    private readonly IPasswordHasher _hasher;
    private readonly IEmailSender _emailSender;

    public UserService(IUserRepository repository, IPasswordHasher hasher, IEmailSender emailSender)
    {
        _repository = repository;
        _hasher = hasher;
        _emailSender = emailSender;
    }

    public async Task RegisterAsync(User user)
    {
        user.PasswordHash = _hasher.Hash(user.Password);
        await _repository.AddAsync(user);
        await _emailSender.SendAsync(user.Email, "Hoşgeldiniz");
    }
}
```

---

## 5. Service Locator Anti-Pattern

❌ **Yanlış Kullanım:** Service Locator ile runtime'da bağımlılık çözmek.

```csharp
public class InvoiceService
{
    public async Task CreateAsync(Invoice invoice)
    {
        var repo = ServiceLocator.Get<IInvoiceRepository>();
        var validator = ServiceLocator.Get<IValidator<Invoice>>();
        var logger = ServiceLocator.Get<ILogger>();

        // Bağımlılıklar gizli, test edilmesi çok zor
    }
}
```

✅ **İdeal Kullanım:** Tüm bağımlılıkları constructor'da açıkça belirtin.

```csharp
public class InvoiceService
{
    private readonly IInvoiceRepository _repository;
    private readonly IValidator<Invoice> _validator;
    private readonly ILogger<InvoiceService> _logger;

    public InvoiceService(
        IInvoiceRepository repository,
        IValidator<Invoice> validator,
        ILogger<InvoiceService> logger)
    {
        _repository = repository;
        _validator = validator;
        _logger = logger;
    }

    public async Task CreateAsync(Invoice invoice)
    {
        await _validator.ValidateAndThrowAsync(invoice);
        await _repository.AddAsync(invoice);
        _logger.LogInformation("Fatura oluşturuldu: {Id}", invoice.Id);
    }
}
```

---

## 6. HttpClient Bağımlılığını Yanlış Yönetmek

❌ **Yanlış Kullanım:** HttpClient'ı doğrudan oluşturmak.

```csharp
public class WeatherService
{
    public async Task<Weather> GetAsync(string city)
    {
        using var client = new HttpClient(); // Her çağrıda yeni socket, port tükenmesi riski
        var response = await client.GetAsync($"https://api.weather.com/{city}");
        return await response.Content.ReadFromJsonAsync<Weather>();
    }
}
```

✅ **İdeal Kullanım:** `IHttpClientFactory` ile yönetin.

```csharp
builder.Services.AddHttpClient<IWeatherService, WeatherService>(client =>
{
    client.BaseAddress = new Uri("https://api.weather.com/");
    client.Timeout = TimeSpan.FromSeconds(10);
});

public class WeatherService : IWeatherService
{
    private readonly HttpClient _client;

    public WeatherService(HttpClient client) => _client = client;

    public async Task<Weather> GetAsync(string city)
    {
        return await _client.GetFromJsonAsync<Weather>(city);
    }
}
```