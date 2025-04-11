# Pipeline Behaviors

Pipeline Behaviors, MediatR pipeline'ında request'lerin işlenmesi sırasında çalışan middleware'lerdir. Cross-cutting concern'ler için idealdir.

## Behavior Özellikleri

1. **Pre-Processing**: Request handler'dan önce çalışır
2. **Post-Processing**: Request handler'dan sonra çalışır
3. **Exception Handling**: Exception'ları yakalayabilir
4. **Response Manipulation**: Response'ları değiştirebilir

## Yaygın Behavior Örnekleri

### Logging Behavior
```csharp
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Handling {RequestName}", typeof(TRequest).Name);
        
        var response = await next();
        
        _logger.LogInformation("Handled {RequestName}", typeof(TRequest).Name);
        
        return response;
    }
}
```

### Validation Behavior
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

### Transaction Behavior
```csharp
public class TransactionBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IUnitOfWork _unitOfWork;

    public TransactionBehavior(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        await _unitOfWork.BeginTransactionAsync();

        try
        {
            var response = await next();
            await _unitOfWork.CommitTransactionAsync();
            return response;
        }
        catch
        {
            await _unitOfWork.RollbackTransactionAsync();
            throw;
        }
    }
}
```

### Performance Behavior
```csharp
public class PerformanceBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<PerformanceBehavior<TRequest, TResponse>> _logger;
    private readonly Stopwatch _timer;

    public PerformanceBehavior(ILogger<PerformanceBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
        _timer = new Stopwatch();
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        _timer.Start();
        
        var response = await next();
        
        _timer.Stop();
        
        var elapsedMilliseconds = _timer.ElapsedMilliseconds;
        
        if (elapsedMilliseconds > 500)
        {
            _logger.LogWarning(
                "Long Running Request: {RequestName} ({ElapsedMilliseconds} milliseconds)",
                typeof(TRequest).Name,
                elapsedMilliseconds);
        }

        return response;
    }
}
```

## Behavior Best Practices

1. **Behavior Sıralaması**
   - Validation
   - Authorization
   - Logging
   - Transaction
   - Caching
   - Performance

2. **Error Handling**
   - Her behavior kendi exception'larını yakalamalı
   - Global exception handling kullanılmalı

3. **Testing**
   - Behavior'lar unit test edilmeli
   - Integration testler yazılmalı

4. **Performance**
   - Behavior'lar hafif olmalı
   - Gereksiz işlemlerden kaçınılmalı

## Behavior Registration

```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    cfg.AddBehavior(typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(LoggingBehavior<,>));
    cfg.AddBehavior(typeof(TransactionBehavior<,>));
    cfg.AddBehavior(typeof(PerformanceBehavior<,>));
});
``` 