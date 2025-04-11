# Error Handling

Error Handling, MediatR'da exception'ları yönetmek için kullanılan bir mekanizmadır. Global exception handling ve özel exception tipleri kullanılır.

## Error Handling Özellikleri

1. **Global Exception Handling**: Tüm exception'lar merkezi olarak yönetilir
2. **Custom Exception Types**: Özel exception tipleri tanımlanır
3. **Error Response**: Hata durumunda özel response'lar dönülür
4. **Logging**: Hatalar loglanır

## Exception Tipleri

### Custom Exceptions
```csharp
public class ValidationException : Exception
{
    public ValidationException(string message) : base(message)
    {
    }

    public ValidationException(string message, Exception innerException) 
        : base(message, innerException)
    {
    }
}

public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message)
    {
    }
}

public class UnauthorizedException : Exception
{
    public UnauthorizedException(string message) : base(message)
    {
    }
}
```

### Exception Middleware
```csharp
public class ExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionMiddleware> _logger;

    public ExceptionMiddleware(
        RequestDelegate next,
        ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An error occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        var response = new ErrorResponse();

        switch (exception)
        {
            case ValidationException validationException:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response = new ErrorResponse
                {
                    StatusCode = StatusCodes.Status400BadRequest,
                    Message = validationException.Message
                };
                break;

            case NotFoundException notFoundException:
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                response = new ErrorResponse
                {
                    StatusCode = StatusCodes.Status404NotFound,
                    Message = notFoundException.Message
                };
                break;

            case UnauthorizedException unauthorizedException:
                context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                response = new ErrorResponse
                {
                    StatusCode = StatusCodes.Status401Unauthorized,
                    Message = unauthorizedException.Message
                };
                break;

            default:
                context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                response = new ErrorResponse
                {
                    StatusCode = StatusCodes.Status500InternalServerError,
                    Message = "Internal server error"
                };
                break;
        }

        await context.Response.WriteAsJsonAsync(response);
    }
}

public class ErrorResponse
{
    public int StatusCode { get; set; }
    public string Message { get; set; }
}
```

## Error Handling Best Practices

1. **Exception Types**
   - Her hata tipi için özel exception sınıfı
   - Exception mesajları açıklayıcı olmalı
   - Inner exception'lar korunmalı

2. **Error Response**
   - HTTP status code'ları doğru kullanılmalı
   - Error response'ları standart olmalı
   - Hata detayları uygun seviyede olmalı

3. **Logging**
   - Tüm hatalar loglanmalı
   - Log seviyeleri doğru kullanılmalı
   - Sensitive data loglanmamalı

4. **Testing**
   - Exception senaryoları test edilmeli
   - Error response'ları test edilmeli
   - Logging test edilmeli

## Error Handling Pipeline

Error handling pipeline üzerinden geçer:

1. Exception Catching
2. Exception Logging
3. Error Response Generation
4. Response Sending

```csharp
public class ErrorHandlingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<ErrorHandlingBehavior<TRequest, TResponse>> _logger;

    public ErrorHandlingBehavior(ILogger<ErrorHandlingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        try
        {
            return await next();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling request {RequestName}", typeof(TRequest).Name);
            throw;
        }
    }
}
```

## Error Handling Registration

```csharp
services.AddMediatR(cfg => {
    cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
    cfg.AddBehavior(typeof(ErrorHandlingBehavior<,>));
});

app.UseMiddleware<ExceptionMiddleware>();
``` 