# Mocking

Mocking, bağımlılıkları izole ederek birim testlerin güvenilir olmasını sağlar; yanlış mock kullanımı kırılgan ve anlamsız testlere yol açar.

---

## 1. Concrete Sınıfı Doğrudan Kullanmak

❌ **Yanlış Kullanım:** Test içinde gerçek bağımlılıkları kullanmak.

```csharp
[Fact]
public async Task CreateOrder_SavesOrder()
{
    var context = new AppDbContext(); // Gerçek veritabanı
    var emailService = new SmtpEmailService(); // Gerçek e-posta gönderir!
    var service = new OrderService(context, emailService);

    await service.CreateAsync(new Order());
}
```

✅ **İdeal Kullanım:** Moq ile bağımlılıkları mock'layın.

```csharp
[Fact]
public async Task CreateOrder_SavesOrderAndSendsEmail()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();
    var service = new OrderService(mockRepo.Object, mockEmail.Object);

    await service.CreateAsync(new Order { CustomerEmail = "test@test.com" });

    mockRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
    mockEmail.Verify(e => e.SendAsync("test@test.com", It.IsAny<string>()), Times.Once);
}
```

---

## 2. Aşırı Detaylı Mock Setup

❌ **Yanlış Kullanım:** İhtiyaç duyulmayan her metodu setup etmek.

```csharp
[Fact]
public async Task GetOrder_ReturnsOrder()
{
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<int>())).ReturnsAsync(new Order());
    mockRepo.Setup(r => r.GetAllAsync()).ReturnsAsync(new List<Order>());
    mockRepo.Setup(r => r.AddAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);
    mockRepo.Setup(r => r.UpdateAsync(It.IsAny<Order>())).Returns(Task.CompletedTask);
    mockRepo.Setup(r => r.DeleteAsync(It.IsAny<int>())).Returns(Task.CompletedTask);
    // Sadece GetByIdAsync kullanılıyor, diğerleri gereksiz

    var service = new OrderService(mockRepo.Object);
    var result = await service.GetAsync(1);

    Assert.NotNull(result);
}
```

✅ **İdeal Kullanım:** Sadece test edilen senaryoya gerekli setup'ları yapın.

```csharp
[Fact]
public async Task GetOrder_WithValidId_ReturnsOrder()
{
    var expectedOrder = new Order { Id = 1, TotalPrice = 500 };
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(expectedOrder);

    var service = new OrderService(mockRepo.Object);

    var result = await service.GetAsync(1);

    Assert.Equal(500, result.TotalPrice);
}
```

---

## 3. Implementation Detail Test Etmek

❌ **Yanlış Kullanım:** İç implementasyonu test edip davranışı test etmemek.

```csharp
[Fact]
public async Task PlaceOrder_CallsRepositoryAndEmailInCorrectOrder()
{
    var sequence = new List<string>();
    var mockRepo = new Mock<IOrderRepository>();
    mockRepo.Setup(r => r.AddAsync(It.IsAny<Order>()))
        .Callback(() => sequence.Add("repo"));
    var mockEmail = new Mock<IEmailService>();
    mockEmail.Setup(e => e.SendAsync(It.IsAny<string>(), It.IsAny<string>()))
        .Callback(() => sequence.Add("email"));

    var service = new OrderService(mockRepo.Object, mockEmail.Object);
    await service.PlaceAsync(new Order());

    Assert.Equal(new[] { "repo", "email" }, sequence); // Sıra değişirse test kırılır
}
```

✅ **İdeal Kullanım:** Davranışı ve sonucu test edin, çağrı sırasını değil.

```csharp
[Fact]
public async Task PlaceOrder_SavesOrderAndNotifiesCustomer()
{
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmail = new Mock<IEmailService>();
    var service = new OrderService(mockRepo.Object, mockEmail.Object);

    await service.PlaceAsync(new Order { CustomerEmail = "test@test.com" });

    mockRepo.Verify(r => r.AddAsync(It.IsAny<Order>()), Times.Once);
    mockEmail.Verify(e => e.SendAsync("test@test.com", It.IsAny<string>()), Times.Once);
}
```

---

## 4. Mock Yerine Stub Kullanılması Gereken Yerlerde Mock Kullanmak

❌ **Yanlış Kullanım:** Basit dönüş değerleri için gereksiz mock framework kullanmak.

```csharp
var mockClock = new Mock<IClock>();
mockClock.Setup(c => c.UtcNow).Returns(new DateTime(2024, 1, 1));
```

✅ **İdeal Kullanım:** Basit senaryolar için lightweight stub oluşturun.

```csharp
public class FakeClock : IClock
{
    public DateTime UtcNow { get; set; }
}

[Fact]
public void IsExpired_WhenPastExpiryDate_ReturnsTrue()
{
    var clock = new FakeClock { UtcNow = new DateTime(2024, 6, 1) };
    var token = new Token { ExpiresAt = new DateTime(2024, 1, 1) };

    Assert.True(token.IsExpired(clock));
}
```

---

## 5. HttpClient'ı Mock'lamak

❌ **Yanlış Kullanım:** HttpClient'ı doğrudan mock'lamaya çalışmak.

```csharp
var mockClient = new Mock<HttpClient>(); // HttpClient mock'lanamaz, sealed metotlar var
```

✅ **İdeal Kullanım:** MockHttpMessageHandler ile HttpClient'ı test edin.

```csharp
public class MockHttpMessageHandler : HttpMessageHandler
{
    private readonly HttpStatusCode _statusCode;
    private readonly string _content;

    public MockHttpMessageHandler(HttpStatusCode statusCode, string content)
    {
        _statusCode = statusCode;
        _content = content;
    }

    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        return Task.FromResult(new HttpResponseMessage
        {
            StatusCode = _statusCode,
            Content = new StringContent(_content)
        });
    }
}

[Fact]
public async Task GetProduct_ReturnsProductFromApi()
{
    var json = JsonSerializer.Serialize(new Product { Id = 1, Name = "Laptop" });
    var handler = new MockHttpMessageHandler(HttpStatusCode.OK, json);
    var client = new HttpClient(handler) { BaseAddress = new Uri("https://api.test.com/") };
    var service = new ProductApiClient(client);

    var result = await service.GetProductAsync(1);

    Assert.Equal("Laptop", result.Name);
}
```