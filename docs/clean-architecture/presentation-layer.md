# Presentation Layer

Presentation katmanı, kullanıcı ile uygulama arasındaki iletişimi yönetir; yanlış kullanımlar controller'larda şişkinlik ve katman ihlallerine neden olur.

---

## 1. Controller İçinde İş Mantığı

❌ **Yanlış Kullanım:** Controller'da veritabanı erişimi ve iş mantığı yazmak.

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    var customer = await _context.Customers.FindAsync(request.CustomerId);
    if (customer == null) return NotFound();

    var order = new Order { CustomerId = customer.Id };
    foreach (var item in request.Items)
    {
        var product = await _context.Products.FindAsync(item.ProductId);
        if (product.Stock < item.Quantity) return BadRequest("Stok yetersiz");
        product.Stock -= item.Quantity;
        order.Items.Add(new OrderItem { ProductId = product.Id, Quantity = item.Quantity });
    }
    _context.Orders.Add(order);
    await _context.SaveChangesAsync();
    return Ok(order);
}
```

✅ **İdeal Kullanım:** Controller sadece HTTP concern'lerini yönetsin.

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request)
{
    var result = await _mediator.Send(new CreateOrderCommand(request.CustomerId, request.Items));
    return CreatedAtAction(nameof(GetOrder), new { id = result }, result);
}
```

---

## 2. Tutarsız API Response Formatı

❌ **Yanlış Kullanım:** Her endpoint farklı response yapısı döndürmek.

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _service.GetAsync(id);
    if (order == null) return NotFound("Sipariş bulunamadı");
    return Ok(order);
}

[HttpGet]
public async Task<IActionResult> GetProducts()
{
    var products = await _service.GetAllAsync();
    return Ok(new { data = products, total = products.Count });
}
```

✅ **İdeal Kullanım:** Standart API response wrapper kullanın.

```csharp
public class ApiResponse<T>
{
    public bool Success { get; init; }
    public T Data { get; init; }
    public string Message { get; init; }
    public List<string> Errors { get; init; }
}

[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _mediator.Send(new GetOrderQuery(id));
    return Ok(new ApiResponse<OrderDto> { Success = true, Data = order });
}

[HttpGet]
public async Task<IActionResult> GetProducts()
{
    var products = await _mediator.Send(new GetProductsQuery());
    return Ok(new ApiResponse<List<ProductDto>> { Success = true, Data = products });
}
```

---

## 3. Exception Handling'i Her Controller'da Tekrarlamak

❌ **Yanlış Kullanım:** Her action'da try-catch yazmak.

```csharp
[HttpPost]
public async Task<IActionResult> CreateProduct(CreateProductDto dto)
{
    try
    {
        var result = await _service.CreateAsync(dto);
        return Ok(result);
    }
    catch (ValidationException ex)
    {
        return BadRequest(ex.Errors);
    }
    catch (NotFoundException ex)
    {
        return NotFound(ex.Message);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Hata oluştu");
        return StatusCode(500, "Bir hata oluştu");
    }
}
```

✅ **İdeal Kullanım:** Global exception handler middleware kullanın.

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new { errors = ex.Errors });
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(new { message = ex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Beklenmeyen hata");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(new { message = "Bir hata oluştu" });
        }
    }
}

// Controller temiz kalır
[HttpPost]
public async Task<IActionResult> CreateProduct(CreateProductDto dto)
{
    var result = await _mediator.Send(new CreateProductCommand(dto));
    return Ok(result);
}
```

---

## 4. Model Binding ve Validation Karışımı

❌ **Yanlış Kullanım:** Controller'da manuel validasyon yapmak.

```csharp
[HttpPost]
public async Task<IActionResult> Register(RegisterRequest request)
{
    if (string.IsNullOrEmpty(request.Email)) return BadRequest("Email boş olamaz");
    if (request.Password.Length < 6) return BadRequest("Şifre en az 6 karakter olmalı");
    if (request.Password != request.ConfirmPassword) return BadRequest("Şifreler eşleşmiyor");

    var result = await _service.RegisterAsync(request);
    return Ok(result);
}
```

✅ **İdeal Kullanım:** FluentValidation ile validasyonu otomatikleştirin.

```csharp
public class RegisterRequestValidator : AbstractValidator<RegisterRequest>
{
    public RegisterRequestValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).MinimumLength(6);
        RuleFor(x => x.ConfirmPassword).Equal(x => x.Password).WithMessage("Şifreler eşleşmiyor");
    }
}

// Controller
[HttpPost]
public async Task<IActionResult> Register(RegisterRequest request)
{
    var result = await _mediator.Send(new RegisterCommand(request));
    return Ok(result);
}
```

---

## 5. Versiyonlama Yapmamak

❌ **Yanlış Kullanım:** API'yi versiyonsuz sunmak.

```csharp
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll() { /* ... */ }
    // Breaking change yapıldığında mevcut istemciler bozulur
}
```

✅ **İdeal Kullanım:** API versiyonlama stratejisi uygulayın.

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});

[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll() { /* v1 response */ }
}

[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/products")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll() { /* v2 response - yeni format */ }
}
```