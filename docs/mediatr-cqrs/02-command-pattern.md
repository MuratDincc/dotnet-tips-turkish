# Command Pattern

Command Pattern, veri değiştirme işlemlerini temsil eden bir pattern'dır. MediatR'da command'lar `IRequest<T>` interface'ini implemente eder.

## Command Özellikleri

1. **Immutable**: Command'lar immutable olmalıdır
2. **Tek Sorumluluk**: Her command tek bir iş yapmalıdır
3. **Validation**: Command'lar validate edilebilir olmalıdır
4. **Idempotent**: Mümkünse idempotent olmalıdır

## Command Örnekleri

### Basit Command
```csharp
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductRepository _repository;

    public CreateProductCommandHandler(IProductRepository repository)
    {
        _repository = repository;
    }

    public async Task<int> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var product = new Product
        {
            Name = request.Name,
            Price = request.Price
        };

        await _repository.AddAsync(product);
        return product.Id;
    }
}
```

### Complex Command
```csharp
public class UpdateProductCommand : IRequest
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Description { get; set; }
}

public class UpdateProductCommandHandler : IRequestHandler<UpdateProductCommand>
{
    private readonly IProductRepository _repository;
    private readonly IUnitOfWork _unitOfWork;

    public UpdateProductCommandHandler(
        IProductRepository repository,
        IUnitOfWork unitOfWork)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
    }

    public async Task<Unit> Handle(UpdateProductCommand request, CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id);
        
        product.Name = request.Name;
        product.Price = request.Price;
        product.Description = request.Description;

        await _repository.UpdateAsync(product);
        await _unitOfWork.SaveChangesAsync();

        return Unit.Value;
    }
}
```

## Command Best Practices

1. **Command Validation**
   - FluentValidation kullanın
   - Validation logic'i behavior'larda olmalı

2. **Command Handler Organization**
   - Her handler tek bir repository kullanmalı
   - Business logic handler içinde olmalı
   - Cross-cutting concern'ler behavior'lara taşınmalı

3. **Error Handling**
   - Özel exception tipleri kullanın
   - Global exception handling kullanın

4. **Testing**
   - Handler'lar unit test edilmeli
   - Integration testler yazılmalı

## Command Pipeline

Command'lar pipeline üzerinden geçer:

1. Validation
2. Authorization
3. Logging
4. Transaction
5. Handler
6. Event Publishing

```csharp
public class CommandPipelineBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // Pre-processing
        var response = await next();
        // Post-processing
        return response;
    }
}
``` 