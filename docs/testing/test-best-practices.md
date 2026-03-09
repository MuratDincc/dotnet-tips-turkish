# Test Best Practices

Test best practices, güvenilir, okunabilir ve sürdürülebilir test süitleri oluşturmayı sağlar; kötü pratikler bakım yükü ve güven kaybına yol açar.

---

## 1. Flaky (Güvenilmez) Test Yazmak

❌ **Yanlış Kullanım:** Zamana veya sıraya bağımlı test yazmak.

```csharp
[Fact]
public void GetRecentOrders_ReturnsLastHourOrders()
{
    var order = new Order { CreatedAt = DateTime.Now.AddMinutes(-30) };
    _context.Orders.Add(order);
    _context.SaveChanges();

    var result = _service.GetRecentOrders();

    Assert.Single(result); // Gece yarısı çalışırsa başarısız olabilir
}
```

✅ **İdeal Kullanım:** Zamanı kontrol altına alarak deterministik test yazın.

```csharp
[Fact]
public void GetRecentOrders_ReturnsLastHourOrders()
{
    var now = new DateTime(2024, 6, 15, 14, 0, 0, DateTimeKind.Utc);
    var clock = new FakeClock(now);
    var order = new Order { CreatedAt = now.AddMinutes(-30) };
    _context.Orders.Add(order);
    _context.SaveChanges();

    var service = new OrderService(_context, clock);
    var result = service.GetRecentOrders();

    Assert.Single(result);
}
```

---

## 2. Magic Number Kullanmak

❌ **Yanlış Kullanım:** Anlamı belirsiz sabit değerler kullanmak.

```csharp
[Fact]
public void Test()
{
    var result = _calculator.Calculate(100, 0.18m, 50, true);
    Assert.Equal(168.2m, result);
}
```

✅ **İdeal Kullanım:** Değişkenlerle anlamı açıkça belirtin.

```csharp
[Fact]
public void Calculate_WithTaxAndShippingForPremiumCustomer_AppliesFreeShipping()
{
    const decimal basePrice = 100m;
    const decimal taxRate = 0.18m;
    const decimal shippingCost = 50m;
    const bool isPremium = true;
    const decimal expectedTotal = 118m; // Premium = ücretsiz kargo

    var result = _calculator.Calculate(basePrice, taxRate, shippingCost, isPremium);

    Assert.Equal(expectedTotal, result);
}
```

---

## 3. FluentAssertions Kullanmamak

❌ **Yanlış Kullanım:** Assert mesajlarının hata durumunda anlamsız olması.

```csharp
[Fact]
public void GetActiveUsers_ReturnsOnlyActiveUsers()
{
    var users = _service.GetActiveUsers();

    Assert.Equal(3, users.Count);
    Assert.True(users.All(u => u.IsActive));
    Assert.Contains(users, u => u.Name == "Ali");
}
// Hata mesajı: "Assert.Equal() Failure. Expected: 3, Actual: 2"
```

✅ **İdeal Kullanım:** FluentAssertions ile okunabilir assert'ler yazın.

```csharp
[Fact]
public void GetActiveUsers_ReturnsOnlyActiveUsers()
{
    var users = _service.GetActiveUsers();

    users.Should().HaveCount(3)
        .And.OnlyContain(u => u.IsActive)
        .And.Contain(u => u.Name == "Ali");
}
// Hata mesajı: "Expected collection to contain 3 item(s), but found 2."
```

---

## 4. Test Coverage Takıntısı

❌ **Yanlış Kullanım:** %100 coverage için anlamsız testler yazmak.

```csharp
[Fact]
public void Constructor_SetsProperties()
{
    var dto = new ProductDto { Id = 1, Name = "Test" };

    Assert.Equal(1, dto.Id);
    Assert.Equal("Test", dto.Name);
}

[Fact]
public void ToString_ReturnsString()
{
    var dto = new ProductDto { Name = "Test" };
    Assert.NotNull(dto.ToString());
}
```

✅ **İdeal Kullanım:** İş mantığı ve kritik yolları test edin, getter/setter test etmeyin.

```csharp
// DTO getter/setter testleri gereksiz
// Bunun yerine gerçek iş mantığını test edin

[Fact]
public void ApplyDiscount_WhenDiscountExceedsPrice_ThrowsDomainException()
{
    var order = new Order { TotalPrice = 100 };

    Assert.Throws<DomainException>(() => order.ApplyDiscount(150));
}

[Fact]
public void ChangeStatus_FromShippedToCancelled_ThrowsDomainException()
{
    var order = new Order { Status = OrderStatus.Shipped };

    var ex = Assert.Throws<DomainException>(() => order.Cancel());
    Assert.Equal("Kargolanmış sipariş iptal edilemez.", ex.Message);
}
```

---

## 5. Shared State ile Test Kirliliği

❌ **Yanlış Kullanım:** Statik veya paylaşılan state kullanmak.

```csharp
public class DiscountTests
{
    private static readonly DiscountService _service = new();
    private static decimal _lastResult;

    [Fact]
    public void Test1()
    {
        _lastResult = _service.Calculate(100);
        Assert.Equal(90, _lastResult);
    }

    [Fact]
    public void Test2()
    {
        Assert.Equal(90, _lastResult); // Test1'e bağımlı
    }
}
```

✅ **İdeal Kullanım:** Her test kendi instance'larını oluştursun.

```csharp
public class DiscountTests
{
    private readonly DiscountService _service;

    public DiscountTests()
    {
        _service = new DiscountService(); // Her test için yeni instance
    }

    [Fact]
    public void Calculate_WithStandardDiscount_ReturnsTenPercentOff()
    {
        var result = _service.Calculate(100);
        Assert.Equal(90, result);
    }

    [Fact]
    public void Calculate_WithZeroAmount_ReturnsZero()
    {
        var result = _service.Calculate(0);
        Assert.Equal(0, result);
    }
}
```