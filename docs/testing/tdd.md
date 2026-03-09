# Test-Driven Development (TDD)

TDD, önce test yazıp ardından kodu geliştirmeyi esas alır; yanlış uygulama gereksiz testlere ve kırılgan test süitlerine yol açar.

---

## 1. Testi Kodu Yazdıktan Sonra Yazmak

❌ **Yanlış Kullanım:** Önce kodu yazıp sonradan test eklemek.

```csharp
// Önce kod yazıldı
public class PriceCalculator
{
    public decimal Calculate(decimal price, decimal taxRate)
    {
        return price + (price * taxRate);
    }
}

// Sonra test yazıldı - implementasyona bağımlı, tasarımı yönlendirmedi
[Fact]
public void Calculate_Test()
{
    var calc = new PriceCalculator();
    Assert.Equal(118, calc.Calculate(100, 0.18m));
}
```

✅ **İdeal Kullanım:** Red-Green-Refactor döngüsünü takip edin.

```csharp
// 1. RED: Önce başarısız test yazın
[Fact]
public void Calculate_WithEighteenPercentTax_ReturnsPriceWithTax()
{
    var calculator = new PriceCalculator();

    var result = calculator.Calculate(100, 0.18m);

    Assert.Equal(118m, result);
}

// 2. GREEN: Testi geçirecek minimum kodu yazın
public class PriceCalculator
{
    public decimal Calculate(decimal price, decimal taxRate)
    {
        return price + (price * taxRate);
    }
}

// 3. REFACTOR: Kodu iyileştirin, testler hala geçmeli
```

---

## 2. Çok Büyük Adımlarla İlerlemek

❌ **Yanlış Kullanım:** Karmaşık senaryoyu tek seferde test etmeye çalışmak.

```csharp
[Fact]
public void PlaceOrder_WithDiscountAndTaxAndShipping_CalculatesCorrectTotal()
{
    var service = new OrderService();
    var order = new Order
    {
        Items = new List<OrderItem>
        {
            new() { Price = 100, Quantity = 2 },
            new() { Price = 50, Quantity = 1 }
        },
        DiscountCode = "SUMMER20",
        ShippingMethod = "express"
    };

    var result = service.PlaceOrder(order);

    Assert.Equal(236.8m, result.Total); // Çok fazla hesaplama, hata bulmak zor
}
```

✅ **İdeal Kullanım:** Küçük adımlarla, her seferinde bir davranışı test edin.

```csharp
[Fact]
public void CalculateSubtotal_WithMultipleItems_ReturnsSumOfPrices()
{
    var calculator = new OrderCalculator();
    var items = new List<OrderItem>
    {
        new() { Price = 100, Quantity = 2 },
        new() { Price = 50, Quantity = 1 }
    };

    var subtotal = calculator.CalculateSubtotal(items);

    Assert.Equal(250m, subtotal);
}

[Fact]
public void ApplyDiscount_WithTwentyPercent_ReducesSubtotal()
{
    var calculator = new OrderCalculator();

    var discounted = calculator.ApplyDiscount(250m, 0.20m);

    Assert.Equal(200m, discounted);
}

[Fact]
public void CalculateTax_WithEighteenPercent_AddsToPrice()
{
    var calculator = new OrderCalculator();

    var withTax = calculator.CalculateTax(200m, 0.18m);

    Assert.Equal(236m, withTax);
}
```

---

## 3. Edge Case'leri Test Etmemek

❌ **Yanlış Kullanım:** Sadece happy path test etmek.

```csharp
[Fact]
public void Divide_TenByTwo_ReturnsFive()
{
    Assert.Equal(5, Calculator.Divide(10, 2));
}
// Sıfıra bölme, negatif sayılar, overflow test edilmemiş
```

✅ **İdeal Kullanım:** Edge case'leri de test edin.

```csharp
[Fact]
public void Divide_TenByTwo_ReturnsFive()
{
    Assert.Equal(5, Calculator.Divide(10, 2));
}

[Fact]
public void Divide_ByZero_ThrowsDivideByZeroException()
{
    Assert.Throws<DivideByZeroException>(() => Calculator.Divide(10, 0));
}

[Fact]
public void Divide_NegativeNumbers_ReturnsPositiveResult()
{
    Assert.Equal(5, Calculator.Divide(-10, -2));
}

[Fact]
public void Divide_ZeroByAny_ReturnsZero()
{
    Assert.Equal(0, Calculator.Divide(0, 5));
}
```

---

## 4. Testleri Birbirine Bağımlı Yazmak

❌ **Yanlış Kullanım:** Testlerin belirli bir sırada çalışmasını gerektirmek.

```csharp
private static int _createdUserId;

[Fact]
public void Test1_CreateUser()
{
    var user = _service.Create("Ali");
    _createdUserId = user.Id; // Diğer testler bu ID'ye bağımlı
}

[Fact]
public void Test2_GetUser()
{
    var user = _service.GetById(_createdUserId); // Test1 çalışmadıysa başarısız
    Assert.Equal("Ali", user.Name);
}
```

✅ **İdeal Kullanım:** Her testi bağımsız ve izole yazın.

```csharp
[Fact]
public void Create_WithValidName_ReturnsUserWithId()
{
    var user = _service.Create("Ali");

    Assert.True(user.Id > 0);
    Assert.Equal("Ali", user.Name);
}

[Fact]
public void GetById_WithExistingUser_ReturnsUser()
{
    var created = _service.Create("Veli");

    var found = _service.GetById(created.Id);

    Assert.Equal("Veli", found.Name);
}
```

---

## 5. Refactor Adımını Atlama

❌ **Yanlış Kullanım:** Green'den sonra refactor yapmadan devam etmek.

```csharp
public decimal CalculateDiscount(Customer customer, decimal amount)
{
    if (customer.Type == "Premium" && amount > 1000) return amount * 0.20m;
    if (customer.Type == "Premium") return amount * 0.10m;
    if (customer.Type == "Gold" && amount > 1000) return amount * 0.15m;
    if (customer.Type == "Gold") return amount * 0.05m;
    return 0;
    // Testler geçiyor ama kod karmaşık, yeni kural eklemek zor
}
```

✅ **İdeal Kullanım:** Testler geçtikten sonra refactor yapın.

```csharp
// Testler hala geçiyor, kod temiz
public decimal CalculateDiscount(Customer customer, decimal amount)
{
    var baseRate = customer.Type switch
    {
        "Premium" => 0.10m,
        "Gold" => 0.05m,
        _ => 0m
    };

    var bonusRate = amount > 1000 ? 0.10m : 0m;

    return amount * (baseRate + bonusRate);
}
```