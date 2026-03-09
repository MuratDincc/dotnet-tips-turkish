# Integration Testing

Integration testler, birden fazla bileşenin birlikte doğru çalıştığını doğrular; yanlış kurulum yavaş, güvenilmez ve bakımı zor testlere yol açar.

---

## 1. Gerçek Veritabanı Kullanmak

❌ **Yanlış Kullanım:** Paylaşılan veritabanına bağlı test yazmak.

```csharp
public class OrderTests
{
    [Fact]
    public async Task CreateOrder_SavesInDatabase()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer("Server=localhost;Database=TestDb;...") // Gerçek DB
            .Options;

        using var context = new AppDbContext(options);
        // Test verileri diğer testlerle çakışabilir
    }
}
```

✅ **İdeal Kullanım:** InMemory veya container-based veritabanı kullanın.

```csharp
public class OrderTests
{
    [Fact]
    public async Task CreateOrder_SavesInDatabase()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString()) // Her test izole
            .Options;

        using var context = new AppDbContext(options);
        var service = new OrderService(context);

        var order = await service.CreateAsync(new CreateOrderDto { CustomerId = 1 });

        Assert.NotEqual(0, order.Id);
        Assert.Equal(1, await context.Orders.CountAsync());
    }
}
```

---

## 2. WebApplicationFactory Kullanmamak

❌ **Yanlış Kullanım:** API testleri için gerçek sunucu başlatmak.

```csharp
[Fact]
public async Task GetProducts_ReturnsOk()
{
    using var client = new HttpClient();
    client.BaseAddress = new Uri("https://localhost:5001"); // Sunucu çalışıyor olmalı

    var response = await client.GetAsync("/api/products");
    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

✅ **İdeal Kullanım:** WebApplicationFactory ile in-memory test sunucusu oluşturun.

```csharp
public class ApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public ApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.RemoveAll<AppDbContext>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetProducts_ReturnsOkWithProducts()
    {
        var response = await _client.GetAsync("/api/products");

        response.EnsureSuccessStatusCode();
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        Assert.NotNull(products);
    }
}
```

---

## 3. Test Verisi Temizliğini Yapmamak

❌ **Yanlış Kullanım:** Testler arasında veri kalıntısı bırakmak.

```csharp
public class ProductTests
{
    private static readonly AppDbContext _context = CreateContext();

    [Fact]
    public async Task Test1_CreateProduct()
    {
        _context.Products.Add(new Product { Name = "Test" });
        await _context.SaveChangesAsync(); // Bu veri Test2'yi etkiler
    }

    [Fact]
    public async Task Test2_GetAllProducts()
    {
        var count = await _context.Products.CountAsync();
        Assert.Equal(0, count); // Başarısız! Test1'in verisi kaldı
    }
}
```

✅ **İdeal Kullanım:** Her test için temiz veritabanı veya transaction rollback kullanın.

```csharp
public class ProductTests : IDisposable
{
    private readonly AppDbContext _context;

    public ProductTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options;
        _context = new AppDbContext(options);
    }

    [Fact]
    public async Task CreateProduct_IncreasesCount()
    {
        _context.Products.Add(new Product { Name = "Test" });
        await _context.SaveChangesAsync();

        Assert.Equal(1, await _context.Products.CountAsync());
    }

    public void Dispose() => _context.Dispose();
}
```

---

## 4. Custom WebApplicationFactory Oluşturmamak

❌ **Yanlış Kullanım:** Her test sınıfında aynı konfigürasyonu tekrarlamak.

```csharp
public class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(b =>
        {
            b.ConfigureServices(s => { /* 20 satır setup */ });
        }).CreateClient();
    }
}

public class ProductApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    public ProductApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(b =>
        {
            b.ConfigureServices(s => { /* Aynı 20 satır setup */ });
        }).CreateClient();
    }
}
```

✅ **İdeal Kullanım:** Paylaşılan custom factory oluşturun.

```csharp
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            services.RemoveAll<AppDbContext>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("IntegrationTests"));

            services.RemoveAll<IEmailService>();
            services.AddSingleton<IEmailService, FakeEmailService>();
        });

        builder.UseEnvironment("Testing");
    }
}

public class OrderApiTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public OrderApiTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task CreateOrder_ReturnsCreated()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { CustomerId = 1, ProductId = 1 });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }
}
```

---

## 5. Testcontainers Kullanmamak

❌ **Yanlış Kullanım:** Integration testlerde InMemory provider ile gerçek DB davranışını simüle etmeye çalışmak.

```csharp
services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("TestDb"));
// InMemory, foreign key, transaction, raw SQL desteklemez
```

✅ **İdeal Kullanım:** Testcontainers ile gerçek veritabanı container'ı kullanın.

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync() => await _container.StartAsync();
    public async Task DisposeAsync() => await _container.DisposeAsync();
}

public class OrderRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly AppDbContext _context;

    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(fixture.ConnectionString)
            .Options;
        _context = new AppDbContext(options);
        _context.Database.EnsureCreated();
    }

    [Fact]
    public async Task Add_WithValidOrder_PersistsToDatabase()
    {
        var repo = new OrderRepository(_context);
        var order = new Order { CustomerId = 1, TotalPrice = 500 };

        await repo.AddAsync(order);
        await _context.SaveChangesAsync();

        var saved = await _context.Orders.FindAsync(order.Id);
        Assert.Equal(500, saved.TotalPrice);
    }
}
```