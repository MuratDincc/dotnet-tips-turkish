# Single Responsibility Principle (SRP)

Her sınıfın yalnızca bir sorumluluğu ve değişmek için yalnızca bir nedeni olmalıdır; aksi halde bakımı zor, test edilemez kod ortaya çıkar.

---

## 1. God Class (Her Şeyi Yapan Sınıf)

❌ **Yanlış Kullanım:** Tek sınıfta birden fazla sorumluluk barındırmak.

```csharp
public class UserService
{
    public void Register(User user)
    {
        // Validasyon
        if (string.IsNullOrEmpty(user.Email)) throw new Exception("Email boş olamaz");

        // Veritabanı kaydı
        _context.Users.Add(user);
        _context.SaveChanges();

        // E-posta gönderimi
        var smtp = new SmtpClient("smtp.server.com");
        smtp.Send(new MailMessage("noreply@app.com", user.Email, "Hoşgeldiniz", "..."));

        // Log kaydı
        File.AppendAllText("log.txt", $"Kullanıcı kaydedildi: {user.Email}");
    }
}
```

✅ **İdeal Kullanım:** Her sorumluluğu ayrı sınıfa taşıyın.

```csharp
public class UserService
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger<UserService> _logger;

    public UserService(IUserRepository repository, IEmailService emailService, ILogger<UserService> logger)
    {
        _repository = repository;
        _emailService = emailService;
        _logger = logger;
    }

    public async Task RegisterAsync(User user)
    {
        await _repository.AddAsync(user);
        await _emailService.SendWelcomeAsync(user.Email);
        _logger.LogInformation("Kullanıcı kaydedildi: {Email}", user.Email);
    }
}
```

---

## 2. Fat Controller

❌ **Yanlış Kullanım:** Controller içinde iş mantığı yazmak.

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(OrderDto dto)
{
    if (dto.Items.Count == 0) return BadRequest("Sipariş boş olamaz");

    var order = new Order { CustomerId = dto.CustomerId };
    decimal total = 0;
    foreach (var item in dto.Items)
    {
        var product = await _context.Products.FindAsync(item.ProductId);
        if (product.Stock < item.Quantity) return BadRequest("Stok yetersiz");
        product.Stock -= item.Quantity;
        total += product.Price * item.Quantity;
    }
    order.TotalPrice = total;
    _context.Orders.Add(order);
    await _context.SaveChangesAsync();
    return Ok(order);
}
```

✅ **İdeal Kullanım:** Controller'ı ince tutun, iş mantığını servis katmanına taşıyın.

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(OrderDto dto)
{
    var result = await _orderService.CreateAsync(dto);
    return Ok(result);
}
```

---

## 3. Birden Fazla Değişim Nedeni Olan Sınıf

❌ **Yanlış Kullanım:** Raporlama sınıfının hem veri çekme hem formatlama hem gönderme yapması.

```csharp
public class ReportGenerator
{
    public Report FetchData(DateTime from, DateTime to)
    {
        // Veritabanından veri çek
    }

    public string FormatAsHtml(Report report)
    {
        // HTML formatla
    }

    public void SendByEmail(string html, string to)
    {
        // E-posta gönder
    }
}
```

✅ **İdeal Kullanım:** Her sorumluluğu kendi sınıfına ayırın.

```csharp
public class ReportDataFetcher
{
    public Task<Report> FetchAsync(DateTime from, DateTime to) { /* ... */ }
}

public class ReportFormatter
{
    public string FormatAsHtml(Report report) { /* ... */ }
}

public class ReportSender
{
    public Task SendByEmailAsync(string content, string to) { /* ... */ }
}
```

---

## 4. Model Sınıfında İş Mantığı ve Persistence

❌ **Yanlış Kullanım:** Entity sınıfına veritabanı işlemleri eklemek.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }

    public void Save()
    {
        using var context = new AppDbContext();
        context.Products.Add(this);
        context.SaveChanges();
    }

    public decimal CalculateDiscount() => Price * 0.10m;
}
```

✅ **İdeal Kullanım:** Entity sadece domain mantığını barındırsın, persistence ayrı olsun.

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }

    public decimal CalculateDiscount() => Price * 0.10m;
}

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public async Task AddAsync(Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
    }
}
```

---

## 5. Servis İçinde Validation ve Mapping

❌ **Yanlış Kullanım:** Validasyon ve mapping işlemlerini servis içine gömmek.

```csharp
public class InvoiceService
{
    public async Task<InvoiceDto> CreateAsync(CreateInvoiceDto dto)
    {
        // Validasyon
        if (dto.Amount <= 0) throw new ArgumentException("Tutar pozitif olmalı");
        if (string.IsNullOrEmpty(dto.CustomerName)) throw new ArgumentException("Müşteri adı boş olamaz");

        var invoice = new Invoice
        {
            Amount = dto.Amount,
            CustomerName = dto.CustomerName,
            CreatedAt = DateTime.UtcNow
        };

        await _context.Invoices.AddAsync(invoice);
        await _context.SaveChangesAsync();

        return new InvoiceDto { Id = invoice.Id, Amount = invoice.Amount };
    }
}
```

✅ **İdeal Kullanım:** Validasyon ve mapping işlemlerini ayrı sorumluluk olarak tanımlayın.

```csharp
public class CreateInvoiceValidator : AbstractValidator<CreateInvoiceDto>
{
    public CreateInvoiceValidator()
    {
        RuleFor(x => x.Amount).GreaterThan(0);
        RuleFor(x => x.CustomerName).NotEmpty();
    }
}

public class InvoiceService
{
    private readonly IInvoiceRepository _repository;
    private readonly IMapper _mapper;

    public async Task<InvoiceDto> CreateAsync(CreateInvoiceDto dto)
    {
        var invoice = _mapper.Map<Invoice>(dto);
        await _repository.AddAsync(invoice);
        return _mapper.Map<InvoiceDto>(invoice);
    }
}
```