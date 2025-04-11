# Testing

Testing, MediatR ve CQRS pattern'ının doğru çalıştığından emin olmak için önemlidir. Unit testler ve integration testler kullanılır.

## Testing Özellikleri

1. **Unit Testing**: Command ve Query handler'ları test edilir
2. **Integration Testing**: Pipeline ve behavior'lar test edilir
3. **Mocking**: Bağımlılıklar mock'lanır
4. **Test Coverage**: Kod coverage'ı ölçülür

## Unit Testing Örnekleri

### Command Handler Test
```csharp
public class CreateProductCommandHandlerTests
{
    private readonly Mock<IProductRepository> _repositoryMock;
    private readonly CreateProductCommandHandler _handler;

    public CreateProductCommandHandlerTests()
    {
        _repositoryMock = new Mock<IProductRepository>();
        _handler = new CreateProductCommandHandler(_repositoryMock.Object);
    }

    [Fact]
    public async Task Handle_ValidCommand_ReturnsProductId()
    {
        // Arrange
        var command = new CreateProductCommand
        {
            Name = "Test Product",
            Price = 100
        };

        var product = new Product
        {
            Id = 1,
            Name = command.Name,
            Price = command.Price
        };

        _repositoryMock
            .Setup(x => x.AddAsync(It.IsAny<Product>()))
            .ReturnsAsync(product);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        Assert.Equal(product.Id, result);
        _repositoryMock.Verify(x => x.AddAsync(It.IsAny<Product>()), Times.Once);
    }

    [Fact]
    public async Task Handle_InvalidCommand_ThrowsValidationException()
    {
        // Arrange
        var command = new CreateProductCommand
        {
            Name = "",
            Price = -1
        };

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(() =>
            _handler.Handle(command, CancellationToken.None));
    }
}
```

### Query Handler Test
```csharp
public class GetProductByIdQueryHandlerTests
{
    private readonly Mock<IProductRepository> _repositoryMock;
    private readonly GetProductByIdQueryHandler _handler;

    public GetProductByIdQueryHandlerTests()
    {
        _repositoryMock = new Mock<IProductRepository>();
        _handler = new GetProductByIdQueryHandler(_repositoryMock.Object);
    }

    [Fact]
    public async Task Handle_ValidQuery_ReturnsProductDto()
    {
        // Arrange
        var query = new GetProductByIdQuery { Id = 1 };

        var product = new Product
        {
            Id = 1,
            Name = "Test Product",
            Price = 100
        };

        _repositoryMock
            .Setup(x => x.GetByIdAsync(query.Id))
            .ReturnsAsync(product);

        // Act
        var result = await _handler.Handle(query, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal(product.Id, result.Id);
        Assert.Equal(product.Name, result.Name);
        Assert.Equal(product.Price, result.Price);
    }

    [Fact]
    public async Task Handle_NonExistentProduct_ThrowsNotFoundException()
    {
        // Arrange
        var query = new GetProductByIdQuery { Id = 1 };

        _repositoryMock
            .Setup(x => x.GetByIdAsync(query.Id))
            .ReturnsAsync((Product)null);

        // Act & Assert
        await Assert.ThrowsAsync<NotFoundException>(() =>
            _handler.Handle(query, CancellationToken.None));
    }
}
```

## Integration Testing Örnekleri

### Pipeline Test
```csharp
public class ValidationPipelineTests
{
    private readonly ServiceProvider _serviceProvider;

    public ValidationPipelineTests()
    {
        var services = new ServiceCollection();
        
        services.AddMediatR(cfg => {
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            cfg.AddBehavior(typeof(ValidationBehavior<,>));
        });
        
        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        
        _serviceProvider = services.BuildServiceProvider();
    }

    [Fact]
    public async Task Send_InvalidCommand_ThrowsValidationException()
    {
        // Arrange
        var mediator = _serviceProvider.GetRequiredService<IMediator>();
        var command = new CreateProductCommand
        {
            Name = "",
            Price = -1
        };

        // Act & Assert
        await Assert.ThrowsAsync<ValidationException>(() =>
            mediator.Send(command));
    }
}
```

## Testing Best Practices

1. **Unit Testing**
   - Her handler için ayrı test sınıfı
   - Tüm senaryolar test edilmeli
   - Mock'lar doğru kullanılmalı

2. **Integration Testing**
   - Pipeline'lar test edilmeli
   - Behavior'lar test edilmeli
   - Gerçek bağımlılıklar kullanılmalı

3. **Test Organization**
   - Test sınıfları organize edilmeli
   - Test metodları açıklayıcı olmalı
   - Test verileri ayrı tutulmalı

4. **Test Coverage**
   - Kod coverage'ı yüksek olmalı
   - Edge case'ler test edilmeli
   - Performance testleri yapılmalı

## Test Helpers

```csharp
public static class TestHelpers
{
    public static Mock<T> CreateMock<T>() where T : class
    {
        return new Mock<T>();
    }

    public static IMediator CreateMediator(ServiceCollection services)
    {
        services.AddMediatR(cfg => {
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
        });
        
        var serviceProvider = services.BuildServiceProvider();
        return serviceProvider.GetRequiredService<IMediator>();
    }

    public static CancellationToken CreateCancellationToken()
    {
        return new CancellationTokenSource().Token;
    }
}
``` 