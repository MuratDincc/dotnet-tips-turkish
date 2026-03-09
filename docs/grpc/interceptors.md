# gRPC Interceptors

Interceptor'lar, gRPC çağrılarına cross-cutting concern'ler ekler; yanlış kullanımlar performans kaybına ve hata izleme zorluğuna yol açar.

---

## 1. Her Servis Metodunda Loglama Tekrarlamak

❌ **Yanlış Kullanım:** Her metoda manuel loglama eklemek.

```csharp
public override async Task<OrderResponse> CreateOrder(CreateOrderRequest request, ServerCallContext context)
{
    _logger.LogInformation("CreateOrder çağrıldı: {@Request}", request);
    var sw = Stopwatch.StartNew();
    var result = await _service.CreateAsync(request);
    _logger.LogInformation("CreateOrder tamamlandı: {Elapsed}ms", sw.ElapsedMilliseconds);
    return result;
}
```

✅ **İdeal Kullanım:** Logging interceptor ile merkezileştirin.

```csharp
public class LoggingInterceptor : Interceptor
{
    private readonly ILogger<LoggingInterceptor> _logger;

    public LoggingInterceptor(ILogger<LoggingInterceptor> logger) => _logger = logger;

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request, ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var method = context.Method;
        _logger.LogInformation("gRPC çağrısı: {Method}", method);
        var sw = Stopwatch.StartNew();

        var response = await continuation(request, context);

        _logger.LogInformation("gRPC tamamlandı: {Method} ({Elapsed}ms)", method, sw.ElapsedMilliseconds);
        return response;
    }
}

// Kayıt
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<LoggingInterceptor>();
});
```

---

## 2. Validation Interceptor Kullanmamak

❌ **Yanlış Kullanım:** Her metoda manuel validasyon eklemek.

```csharp
public override async Task<UserResponse> CreateUser(CreateUserRequest request, ServerCallContext context)
{
    if (string.IsNullOrEmpty(request.Email))
        throw new RpcException(new Status(StatusCode.InvalidArgument, "Email boş olamaz"));
    if (string.IsNullOrEmpty(request.Name))
        throw new RpcException(new Status(StatusCode.InvalidArgument, "Ad boş olamaz"));
    // ...
}
```

✅ **İdeal Kullanım:** Validation interceptor ile merkezileştirin.

```csharp
public class ValidationInterceptor : Interceptor
{
    private readonly IServiceProvider _serviceProvider;

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request, ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        var validator = _serviceProvider.GetService<IValidator<TRequest>>();
        if (validator != null)
        {
            var result = await validator.ValidateAsync(request);
            if (!result.IsValid)
            {
                var errors = string.Join("; ", result.Errors.Select(e => e.ErrorMessage));
                throw new RpcException(new Status(StatusCode.InvalidArgument, errors));
            }
        }

        return await continuation(request, context);
    }
}
```

---

## 3. Client Interceptor ile Retry

❌ **Yanlış Kullanım:** Her çağrıda manuel retry yazmak.

```csharp
OrderResponse response = null;
for (int i = 0; i < 3; i++)
{
    try
    {
        response = await _client.GetOrderAsync(request);
        break;
    }
    catch (RpcException ex) when (ex.StatusCode == StatusCode.Unavailable)
    {
        await Task.Delay(1000 * (i + 1));
    }
}
```

✅ **İdeal Kullanım:** gRPC retry policy veya client interceptor kullanın.

```csharp
// gRPC built-in retry policy
builder.Services.AddGrpcClient<OrderService.OrderServiceClient>(options =>
{
    options.Address = new Uri("https://order-service:5001");
})
.ConfigureChannel(options =>
{
    options.ServiceConfig = new ServiceConfig
    {
        MethodConfigs =
        {
            new MethodConfig
            {
                Names = { MethodName.Default },
                RetryPolicy = new RetryPolicy
                {
                    MaxAttempts = 3,
                    InitialBackoff = TimeSpan.FromSeconds(1),
                    MaxBackoff = TimeSpan.FromSeconds(5),
                    BackoffMultiplier = 2,
                    RetryableStatusCodes = { StatusCode.Unavailable }
                }
            }
        }
    };
});
```

---

## 4. Metadata ile Context Bilgisi Taşımamak

❌ **Yanlış Kullanım:** Correlation ID gibi bilgileri request message'a eklemek.

```protobuf
message CreateOrderRequest {
    string correlation_id = 1;  // Her message'a eklenmek zorunda
    string trace_id = 2;
    int32 customer_id = 3;
    // ...
}
```

✅ **İdeal Kullanım:** Metadata (headers) ile cross-cutting bilgileri taşıyın.

```csharp
// Client interceptor
public class CorrelationIdInterceptor : Interceptor
{
    public override AsyncUnaryCall<TResponse> AsyncUnaryCall<TRequest, TResponse>(
        TRequest request, ClientInterceptorContext<TRequest, TResponse> context,
        AsyncUnaryCallContinuation<TRequest, TResponse> continuation)
    {
        var headers = context.Options.Headers ?? new Metadata();
        headers.Add("x-correlation-id", Activity.Current?.Id ?? Guid.NewGuid().ToString());

        var newContext = new ClientInterceptorContext<TRequest, TResponse>(
            context.Method, context.Host,
            context.Options.WithHeaders(headers));

        return continuation(request, newContext);
    }
}

// Server tarafında okuma
var correlationId = context.RequestHeaders.GetValue("x-correlation-id");
```

---

## 5. Health Check Eklemememek

❌ **Yanlış Kullanım:** gRPC servisinin sağlığını izlememek.

```csharp
app.MapGrpcService<OrderService>();
// Servis sağlığı bilinmiyor, orchestrator kesintileri tespit edemez
```

✅ **İdeal Kullanım:** gRPC Health Check protokolünü uygulayın.

```csharp
builder.Services.AddGrpcHealthChecks()
    .AddCheck("database", () =>
    {
        // DB bağlantısını kontrol et
        return _context.Database.CanConnect()
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Unhealthy();
    });

app.MapGrpcService<OrderService>();
app.MapGrpcHealthChecksService();
```