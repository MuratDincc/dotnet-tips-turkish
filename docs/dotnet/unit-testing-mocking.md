# Unit Testing ve Mocking

Unit testing ve mocking, doğru uygulandığında yazılım projelerinin güvenilirliğini artırır. Ancak bu araçların yanlış kullanımı, hem test süreçlerini karmaşıklaştırır hem de kodun kalitesini olumsuz etkiler. İşte sık yapılan hatalar ve doğru kullanım örnekleri.

---

## 1. Tek Davranışı Test Etmek

❌ **Yanlış Kullanım:** Bir testin birden fazla davranışı doğrulamaya çalışması, testlerin amacını karmaşıklaştırır.

```csharp
[Fact]
public void AddAndSubtract_ShouldReturnCorrectResults()
{
    var calculator = new Calculator();
    var addResult = calculator.Add(2, 3);
    var subtractResult = calculator.Subtract(5, 3);

    Assert.Equal(5, addResult);
    Assert.Equal(2, subtractResult);
}
```

✅ **İdeal Kullanım:** Her davranışı ayrı bir testte doğrulayın.

```csharp
[Fact]
public void Add_ShouldReturnSum()
{
    var calculator = new Calculator();
    var result = calculator.Add(2, 3);
    Assert.Equal(5, result);
}

[Fact]
public void Subtract_ShouldReturnDifference()
{
    var calculator = new Calculator();
    var result = calculator.Subtract(5, 3);
    Assert.Equal(2, result);
}
```

---

## 2. Mocklama ile Aşırı Karmaşıklık

❌ **Yanlış Kullanım:** Mocklama, yalnızca gereksiz bir soyutlama katmanı ekliyorsa faydasızdır.

```csharp
var mockCalculator = new Mock<ICalculator>();
mockCalculator.Setup(calc => calc.Add(2, 3)).Returns(5);

var result = mockCalculator.Object.Add(2, 3);

Assert.Equal(5, result); // Mock burada gereksiz.
```

✅ **İdeal Kullanım:** Mocklama, yalnızca bağımlılıkları izole etmek için kullanılmalıdır.

```csharp
var mockWeatherService = new Mock<IWeatherService>();
mockWeatherService.Setup(service => service.GetWeather()).Returns("Güneşli");

var reporter = new WeatherReporter(mockWeatherService.Object);
var result = reporter.Report();

Assert.Equal("Güneşli", result);
```
---

## 3. Testler Arasında Bağımlılık Oluşturmak

❌ **Yanlış Kullanım:** Bir test, başka bir testin sonuçlarına bağımlı olmamalıdır.

```csharp
[Fact]
public void Test_AddAndSubtract()
{
    var calculator = new Calculator();
    var sum = calculator.Add(2, 3);
    var difference = calculator.Subtract(sum, 2); // Bu test, Add metoduna bağımlıdır.

    Assert.Equal(3, difference);
}
```

✅ **İdeal Kullanım:** Testler bağımsız olmalı ve birbirlerinden etkilenmemelidir.

```csharp
[Fact]
public void Add_ShouldReturnSum()
{
    var calculator = new Calculator();
    var result = calculator.Add(2, 3);
    Assert.Equal(5, result);
}

[Fact]
public void Subtract_ShouldReturnDifference()
{
    var calculator = new Calculator();
    var result = calculator.Subtract(5, 2);
    Assert.Equal(3, result);
}
```

---

## 4. Assert Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Test sonuçlarını sadece konsola yazdırmak yetersizdir.

```csharp
[Fact]
public void Add_ShouldPrintResult()
{
    var calculator = new Calculator();
    var result = calculator.Add(2, 3);
    Console.WriteLine(result); // Konsol çıktısı yeterli değildir.
}
```

✅ **İdeal Kullanım:** `Assert` ifadeleri ile sonuçları doğrulayın.

```csharp
[Fact]
public void Add_ShouldReturnSum()
{
    var calculator = new Calculator();
    var result = calculator.Add(2, 3);
    Assert.Equal(5, result);
}
```

---

## 5. Parametreli Testler Kullanmamak

🔴 **Yanlış Kullanım:** Her veri seti için ayrı test yazmak gereksiz tekrara neden olur.

```csharp
[Fact]
public void Add_ShouldReturn5()
{
    var calculator = new Calculator();
    var result = calculator.Add(2, 3);
    Assert.Equal(5, result);
}

[Fact]
public void Add_ShouldReturnNegative2()
{
    var calculator = new Calculator();
    var result = calculator.Add(-1, -1);
    Assert.Equal(-2, result);
}
```

✅ **İdeal Kullanım:** Parametreli testlerle tekrarı azaltın.

```csharp
[Theory]
[InlineData(2, 3, 5)]
[InlineData(-1, -1, -2)]
[InlineData(0, 0, 0)]
public void Add_ShouldReturnSum(int a, int b, int expected)
{
    var calculator = new Calculator();
    var result = calculator.Add(a, b);
    Assert.Equal(expected, result);
}
```

---

## 6. Mock Nesneleri Yanlış Konumlandırmak

🔴 **Yanlış Kullanım:** Mock nesnelerini metod seviyesinde oluşturmak kod karmaşıklığını artırır.

```csharp
[Fact]
public void Report_ShouldReturnWeather()
{
    var mockWeatherService = new Mock<IWeatherService>(); // Mock burada oluşturulmuş
    mockWeatherService.Setup(service => service.GetWeather()).Returns("Güneşli");

    var reporter = new WeatherReporter(mockWeatherService.Object);
    var result = reporter.Report();

    Assert.Equal("Güneşli", result);
}
```

✅ **İdeal Kullanım:** Mock nesnelerini sınıf seviyesinde yapılandırın.

```csharp
public class WeatherReporterTests
{
    private readonly Mock<IWeatherService> _mockWeatherService;
    private readonly WeatherReporter _reporter;

    public WeatherReporterTests()
    {
        _mockWeatherService = new Mock<IWeatherService>();
        _reporter = new WeatherReporter(_mockWeatherService.Object);
    }

    [Fact]
    public void Report_ShouldReturnWeather()
    {
        _mockWeatherService.Setup(service => service.GetWeather()).Returns("Güneşli");
        var result = _reporter.Report();
        Assert.Equal("Güneşli", result);
    }
}
```