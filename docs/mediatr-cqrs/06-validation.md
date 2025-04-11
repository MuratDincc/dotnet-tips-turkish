# Validation

Validation, MediatR'da request'lerin doğruluğunu kontrol etmek için kullanılan bir mekanizmadır. FluentValidation kütüphanesi ile entegre çalışır.

## Validation Özellikleri

1. **Request Validation**: Command ve Query'lerin doğruluğunu kontrol eder
2. **Custom Validation**: Özel validation kuralları tanımlanabilir
3. **Cross-Cutting**: Validation logic'i merkezi olarak yönetilir
4. **Error Handling**: Validation hataları özel exception'lar ile yönetilir

## Validation Örnekleri

### Command Validation
```csharp
public class CreateProductCommandValidator : AbstractValidator<CreateProductCommand>
{
    public CreateProductCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.Price)
            .GreaterThan(0);

        RuleFor(x => x.Description)
            .MaximumLength(500);
    }
}

public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string Description { get; set; }
}
```

### Query Validation
```csharp
public class GetProductsQueryValidator : AbstractValidator<GetProductsQuery>
{
    public GetProductsQueryValidator()
    {
        RuleFor(x => x.PageNumber)
            .GreaterThan(0);

        RuleFor(x => x.PageSize)
            .GreaterThan(0)
            .LessThanOrEqualTo(100);

        RuleFor(x => x.SortBy)
            .Must(BeAValidSortField)
            .When(x => !string.IsNullOrEmpty(x.SortBy));
    }

    private bool BeAValidSortField(string sortBy)
    {
        var validFields = new[] { "name", "price", "createdDate" };
        return validFields.Contains(sortBy.ToLower());
    }
}

public class GetProductsQuery : IRequest<PagedList<ProductDto>>
{
    public int PageNumber { get; set; } = 1;
    public int PageSize { get; set; } = 10;
    public string SortBy { get; set; }
    public bool SortDescending { get; set; }
}
```

### Custom Validation
```csharp
public class UniqueProductNameValidator : AbstractValidator<CreateProductCommand>
{
    private readonly IProductRepository _repository;

    public UniqueProductNameValidator(IProductRepository repository)
    {
        _repository = repository;

        RuleFor(x => x.Name)
            .MustAsync(BeUniqueName)
            .WithMessage("Product name must be unique");
    }

    private async Task<bool> BeUniqueName(string name, CancellationToken cancellationToken)
    {
        return !await _repository.ExistsAsync(x => x.Name == name);
    }
}
```

## Validation Best Practices

1. **Validation Rules**
   - Her request için ayrı validator sınıfı
   - Validation kuralları açık ve anlaşılır olmalı
   - Custom validation kuralları kullanılmalı

2. **Error Messages**
   - Hata mesajları açıklayıcı olmalı
   - Çoklu dil desteği olmalı
   - Hata kodları kullanılmalı

3. **Performance**
   - Validation kuralları optimize edilmeli
   - Gereksiz validation'lardan kaçınılmalı
   - Async validation kullanılmalı

4. **Testing**
   - Validator'lar unit test edilmeli
   - Edge case'ler test edilmeli
   - Custom validation'lar test edilmeli

## Validation Pipeline

Validation pipeline üzerinden geçer:

1. Rule Definition
2. Rule Execution
3. Error Collection
4. Exception Throwing

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var context = new ValidationContext<TRequest>(request);
        
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(result => result.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Any())
        {
            throw new ValidationException(failures);
        }

        return await next();
    }
}
```

## Validation Registration

```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    cfg.AddBehavior(typeof(ValidationBehavior<,>));
});

services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
``` 