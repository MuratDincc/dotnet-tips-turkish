# MediatR ve CQRS

MediatR ve CQRS (Command Query Responsibility Segregation) pattern'ı, modern .NET uygulamalarında yaygın olarak kullanılan güçlü bir mimari yaklaşımdır. Bu rehber, bu pattern'ların etkili kullanımına dair ipuçları ve best practice'leri içermektedir.

## İçindekiler

- [Temel Kavramlar](01-basic-concepts.md)
- [Command Pattern](02-command-pattern.md)
- [Query Pattern](03-query-pattern.md)
- [Pipeline Behaviors](04-pipeline-behaviors.md)
- [Event Handling](05-event-handling.md)
- [Validation](06-validation.md)
- [Error Handling](07-error-handling.md)
- [Testing](08-testing.md)
- [Best Practices](09-best-practices.md)

Her bölüm, konunun detaylı bir şekilde anlatımını ve pratik örneklerini içermektedir.

## Temel Kavramlar

### MediatR Nedir?
MediatR, in-process mesajlaşma için basit bir mediator pattern implementasyonudur. Request/response ve notification pattern'larını destekler.

### CQRS Nedir?
CQRS, veri okuma (Query) ve yazma (Command) işlemlerini birbirinden ayıran bir pattern'dır. Bu ayrım, uygulamanın ölçeklenebilirliğini ve bakımını kolaylaştırır.

## Pratik İpuçları

### 1. Command ve Query Ayrımı
```csharp
// Command örneği
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Query örneği
public class GetProductByIdQuery : IRequest<ProductDto>
{
    public int Id { get; set; }
}
```

### 2. Pipeline Behavior Kullanımı
```csharp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // Request öncesi loglama
        var response = await next();
        // Response sonrası loglama
        return response;
    }
}
```

### 3. Event Handling
```csharp
public class ProductCreatedEvent : INotification
{
    public int ProductId { get; set; }
}

public class ProductCreatedEventHandler : INotificationHandler<ProductCreatedEvent>
{
    public async Task Handle(ProductCreatedEvent notification, CancellationToken cancellationToken)
    {
        // Event handling logic
    }
}
```

## Best Practices

1. **Command ve Query Sınıfları**
   - Command'lar immutable olmalı
   - Query'ler sadece veri okumalı
   - Her command/query tek bir iş yapmalı

2. **Handler Organizasyonu**
   - Her handler tek bir sorumluluğa sahip olmalı
   - Business logic handler içinde olmalı
   - Cross-cutting concern'ler behavior'lara taşınmalı

3. **Validation**
   - FluentValidation kullanımı önerilir
   - Validation logic'i behavior'larda olmalı

4. **Error Handling**
   - Global exception handling kullanılmalı
   - Özel exception tipleri tanımlanmalı

## Örnek Proje Yapısı

```
Project/
├── Commands/
│   ├── CreateProduct/
│   │   ├── CreateProductCommand.cs
│   │   ├── CreateProductCommandHandler.cs
│   │   └── CreateProductCommandValidator.cs
│   └── UpdateProduct/
├── Queries/
│   ├── GetProductById/
│   │   ├── GetProductByIdQuery.cs
│   │   └── GetProductByIdQueryHandler.cs
│   └── GetProducts/
├── Events/
│   ├── ProductCreated/
│   │   ├── ProductCreatedEvent.cs
│   │   └── ProductCreatedEventHandler.cs
│   └── ProductUpdated/
└── Behaviors/
    ├── LoggingBehavior.cs
    └── ValidationBehavior.cs
``` 