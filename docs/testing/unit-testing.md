# Unit Testing

Unit testler, kodun en küçük birimlerini izole ederek doğrular; yanlış yazılan testler güvenilmez sonuçlara ve bakım yüküne yol açar.

---

## 1. Test İsimlendirmesini Anlamsız Bırakmak

❌ **Yanlış Kullanım:** Ne test edildiği anlaşılmayan isimler kullanmak.

```csharp
[Fact]
public void Test1()
{
    var service = new OrderService();
    var result = service.Calculate(100, 0.1m);
    Assert.Equal(90, result);
}
```

✅ **İdeal Kullanım:** MethodName_Scenario_ExpectedResult formatını kullanın.

```csharp
[Fact]
public void Calculate_WithTenPercentDiscount_ReturnsDiscountedPrice()
{
    var service = new OrderService();

    var result = service.Calculate(100, 0.1m);

    Assert.Equal(90, result);
}
```

---

## 2. Arrange-Act-Assert Yapısını Kullanmamak

❌ **Yanlış Kullanım:** Test adımlarını karıştırmak.

```csharp
[Fact]
public void CreateOrder_Test()
{
    var order = new Order();
    order.AddItem(new Product("Laptop", 5000));
    Assert.Equal(1, order.Items.Count);
    order.AddItem(new Product("Mouse", 200));
    Assert.Equal(2, order.Items.Count);
    Assert.Equal(5200, order.TotalPrice);
}
```

✅ **İdeal Kullanım:** Arrange, Act, Assert bölümlerini net ayırın.

```csharp
[Fact]
public void AddItem_WhenTwoItemsAdded_CalculatesTotalCorrectly()
{
    // Arrange
    var order = new Order();
    var laptop = new Product("Laptop", 5000);
    var mouse = new Product("Mouse", 200);

    // Act
    order.AddItem(laptop);
    order.AddItem(mouse);

    // Assert
    Assert.Equal(2, order.Items.Count);
    Assert.Equal(5200, order.TotalPrice);
}
```

---

## 3. Birden Fazla Şeyi Test Etmek

❌ **Yanlış Kullanım:** Tek testte birden fazla davranışı doğrulamak.

```csharp
[Fact]
public void UserService_Tests()
{
    var service = new UserService(_mockRepo.Object);

    // Kullanıcı oluşturma testi
    var user = service.Create("Ali", "ali@test.com");
    Assert.NotNull(user);

    // Kullanıcı güncelleme testi
    user.Name = "Veli";
    service.Update(user);
    Assert.Equal("Veli", user.Name);

    // Kullanıcı silme testi
    service.Delete(user.Id);
    Assert.Null(service.GetById(user.Id));
}
```

✅ **İdeal Kullanım:** Her test tek bir davranışı doğrulasın.

```csharp
[Fact]
public void Create_WithValidData_ReturnsUser()
{
    var service = new UserService(_mockRepo.Object);

    var user = service.Create("Ali", "ali@test.com");

    Assert.NotNull(user);
    Assert.Equal("Ali", user.Name);
}

[Fact]
public void Delete_WithExistingUser_RemovesFromRepository()
{
    var service = new UserService(_mockRepo.Object);

    service.Delete(1);

    _mockRepo.Verify(r => r.Delete(1), Times.Once);
}
```

---

## 4. Test Verisi Oluşturmayı Karmaşıklaştırmak

❌ **Yanlış Kullanım:** Her testte uzun nesne oluşturma kodu tekrarlamak.

```csharp
[Fact]
public void CalculateShipping_ForPremiumCustomer_ReturnsFreeShipping()
{
    var customer = new Customer
    {
        Id = 1, Name = "Ali", Email = "ali@test.com",
        Type = CustomerType.Premium, CreatedAt = DateTime.UtcNow,
        Address = new Address { City = "Istanbul", Country = "TR", ZipCode = "34000" }
    };
    var order = new Order
    {
        Id = 1, CustomerId = 1, TotalPrice = 500,
        Items = new List<OrderItem> { new() { ProductId = 1, Quantity = 2, Price = 250 } }
    };
    // ... test mantığı
}
```

✅ **İdeal Kullanım:** Test Data Builder ile temiz test verisi oluşturun.

```csharp
public class CustomerBuilder
{
    private CustomerType _type = CustomerType.Standard;

    public CustomerBuilder AsPremium() { _type = CustomerType.Premium; return this; }

    public Customer Build() => new()
    {
        Id = 1, Name = "Test User", Email = "test@test.com",
        Type = _type, CreatedAt = DateTime.UtcNow,
        Address = new Address { City = "Istanbul", Country = "TR", ZipCode = "34000" }
    };
}

[Fact]
public void CalculateShipping_ForPremiumCustomer_ReturnsFreeShipping()
{
    var customer = new CustomerBuilder().AsPremium().Build();
    var service = new ShippingService();

    var cost = service.CalculateShipping(customer);

    Assert.Equal(0, cost);
}
```

---

## 5. Parametreli Testleri Tekrarlamak

❌ **Yanlış Kullanım:** Farklı girdiler için aynı testi kopyalamak.

```csharp
[Fact]
public void IsValid_WithEmptyEmail_ReturnsFalse()
{
    Assert.False(EmailValidator.IsValid(""));
}

[Fact]
public void IsValid_WithNoAtSign_ReturnsFalse()
{
    Assert.False(EmailValidator.IsValid("test.com"));
}

[Fact]
public void IsValid_WithNoDomain_ReturnsFalse()
{
    Assert.False(EmailValidator.IsValid("test@"));
}
```

✅ **İdeal Kullanım:** `[Theory]` ve `[InlineData]` ile parametreli test yazın.

```csharp
[Theory]
[InlineData("")]
[InlineData("test.com")]
[InlineData("test@")]
[InlineData("@domain.com")]
[InlineData("test @domain.com")]
public void IsValid_WithInvalidEmail_ReturnsFalse(string email)
{
    Assert.False(EmailValidator.IsValid(email));
}

[Theory]
[InlineData("test@domain.com")]
[InlineData("user.name@domain.co")]
public void IsValid_WithValidEmail_ReturnsTrue(string email)
{
    Assert.True(EmailValidator.IsValid(email));
}
```

---

## 6. Exception Testlerini Yanlış Yazmak

❌ **Yanlış Kullanım:** Try-catch ile exception test etmek.

```csharp
[Fact]
public void Withdraw_InsufficientFunds_ThrowsException()
{
    try
    {
        var account = new BankAccount(100);
        account.Withdraw(200);
        Assert.True(false, "Exception bekleniyor");
    }
    catch (InsufficientFundsException)
    {
        Assert.True(true);
    }
}
```

✅ **İdeal Kullanım:** `Assert.Throws` ile temiz exception testi yazın.

```csharp
[Fact]
public void Withdraw_WhenAmountExceedsBalance_ThrowsInsufficientFundsException()
{
    var account = new BankAccount(100);

    var exception = Assert.Throws<InsufficientFundsException>(
        () => account.Withdraw(200));

    Assert.Equal("Yetersiz bakiye.", exception.Message);
}
```