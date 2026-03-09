# Application Layer

Application katmanı, use case'leri orkestre eder ve domain ile altyapı arasında köprü kurar; yanlış kullanımlar iş mantığının yanlış katmana yerleşmesine neden olur.

---

## 1. Application Katmanında İş Mantığı Yazmak

❌ **Yanlış Kullanım:** Domain kurallarını application servisinde uygulamak.

```csharp
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    public async Task<int> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = new Order();
        decimal total = 0;

        foreach (var item in request.Items)
        {
            if (item.Quantity <= 0) throw new ValidationException("Miktar pozitif olmalı");
            total += item.Price * item.Quantity;
        }

        if (total > 50000) throw new BusinessException("Sipariş limiti aşıldı");

        order.TotalPrice = total;
        order.Status = OrderStatus.Pending;
        // ...
    }
}
```

✅ **İdeal Kullanım:** İş mantığını domain'de tutun, application sadece orkestre etsin.

```csharp
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public async Task<int> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);

        foreach (var item in request.Items)
        {
            order.AddItem(item.ProductId, item.Price, item.Quantity); // Domain kuralları burada uygulanır
        }

        await _repository.AddAsync(order);
        await _unitOfWork.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

---

## 2. DTO ve Domain Model Karışımı

❌ **Yanlış Kullanım:** Domain entity'sini doğrudan API response olarak döndürmek.

```csharp
[HttpGet("{id}")]
public async Task<Order> GetOrder(int id)
{
    return await _context.Orders
        .Include(o => o.Items)
        .Include(o => o.Customer)
        .FirstOrDefaultAsync(o => o.Id == id);
    // Circular reference, gereksiz veri, domain detayları sızar
}
```

✅ **İdeal Kullanım:** DTO ile domain model'i ayırın.

```csharp
public record OrderDto(int Id, string CustomerName, decimal TotalPrice, List<OrderItemDto> Items);
public record OrderItemDto(string ProductName, int Quantity, decimal Price);

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    public async Task<OrderDto> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await _repository.GetByIdAsync(request.OrderId);
        if (order == null) throw new NotFoundException("Sipariş bulunamadı.");

        return new OrderDto(
            order.Id,
            order.Customer.Name,
            order.TotalPrice,
            order.Items.Select(i => new OrderItemDto(i.ProductName, i.Quantity, i.Price)).ToList()
        );
    }
}
```

---

## 3. Cross-Cutting Concern'leri Handler İçine Gömmek

❌ **Yanlış Kullanım:** Her handler'da loglama, validasyon, transaction tekrarlamak.

```csharp
public class CreateProductHandler : IRequestHandler<CreateProductCommand, int>
{
    public async Task<int> Handle(CreateProductCommand request, CancellationToken ct)
    {
        _logger.LogInformation("CreateProduct başlatıldı");

        var validationResult = await _validator.ValidateAsync(request);
        if (!validationResult.IsValid) throw new ValidationException(validationResult.Errors);

        using var transaction = await _context.Database.BeginTransactionAsync();
        // ... iş mantığı
        await transaction.CommitAsync();

        _logger.LogInformation("CreateProduct tamamlandı");
        return product.Id;
    }
}
```

✅ **İdeal Kullanım:** Pipeline Behavior ile cross-cutting concern'leri merkezileştirin.

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = _validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Handler temiz kalır
public class CreateProductHandler : IRequestHandler<CreateProductCommand, int>
{
    public async Task<int> Handle(CreateProductCommand request, CancellationToken ct)
    {
        var product = new Product(request.Name, request.Price);
        await _repository.AddAsync(product);
        await _unitOfWork.SaveChangesAsync(ct);
        return product.Id;
    }
}
```

---

## 4. Application Katmanında Altyapı Bağımlılığı

❌ **Yanlış Kullanım:** Application katmanında concrete altyapı sınıflarını kullanmak.

```csharp
public class SendNotificationHandler : IRequestHandler<SendNotificationCommand>
{
    private readonly SmtpClient _smtp;
    private readonly TwilioClient _twilio;

    public async Task Handle(SendNotificationCommand request, CancellationToken ct)
    {
        await _smtp.SendMailAsync(new MailMessage(/* ... */));
        await _twilio.SendSmsAsync(/* ... */);
    }
}
```

✅ **İdeal Kullanım:** Application katmanında interface tanımlayın, implementasyonu Infrastructure'a bırakın.

```csharp
// Application Layer
public interface INotificationService
{
    Task SendEmailAsync(string to, string subject, string body);
    Task SendSmsAsync(string to, string message);
}

public class SendNotificationHandler : IRequestHandler<SendNotificationCommand>
{
    private readonly INotificationService _notificationService;

    public async Task Handle(SendNotificationCommand request, CancellationToken ct)
    {
        await _notificationService.SendEmailAsync(request.Email, request.Subject, request.Body);
    }
}

// Infrastructure Layer
public class NotificationService : INotificationService
{
    public async Task SendEmailAsync(string to, string subject, string body) { /* SMTP */ }
    public async Task SendSmsAsync(string to, string message) { /* Twilio */ }
}
```

---

## 5. Mapping Mantığını Dağıtmak

❌ **Yanlış Kullanım:** Her yerde manuel mapping yapmak.

```csharp
public class GetProductHandler : IRequestHandler<GetProductQuery, ProductDto>
{
    public async Task<ProductDto> Handle(GetProductQuery request, CancellationToken ct)
    {
        var product = await _repository.GetByIdAsync(request.Id);
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            CategoryName = product.Category.Name,
            // 20 satır daha mapping...
        };
    }
}
```

✅ **İdeal Kullanım:** Mapping profillerini merkezi olarak tanımlayın.

```csharp
public class ProductMappingProfile : Profile
{
    public ProductMappingProfile()
    {
        CreateMap<Product, ProductDto>()
            .ForMember(d => d.CategoryName, opt => opt.MapFrom(s => s.Category.Name));
    }
}

public class GetProductHandler : IRequestHandler<GetProductQuery, ProductDto>
{
    private readonly IMapper _mapper;
    private readonly IProductRepository _repository;

    public async Task<ProductDto> Handle(GetProductQuery request, CancellationToken ct)
    {
        var product = await _repository.GetByIdAsync(request.Id);
        return _mapper.Map<ProductDto>(product);
    }
}
```