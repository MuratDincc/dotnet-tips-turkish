# Request Sampling ve Log Filtreleme Teknikleri

Request sampling ve log filtreleme, telemetri maliyetini kontrol altında tutarken önemli verilerin kaybolmasını önler; yanlış yapılandırma ya aşırı veri ya da eksik izleme sorununa yol açar.

---

## 1. Tüm İstekleri Trace Etmek

❌ **Yanlış Kullanım:** Production'da %100 sampling ile çalışmak.

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetSampler(new AlwaysOnSampler()));
// Yüksek trafik: günde milyonlarca trace, backend maliyeti çok yüksek
```

✅ **İdeal Kullanım:** Akıllı sampling stratejisi uygulayın.

```csharp
public class SmartSampler : Sampler
{
    private readonly TraceIdRatioBasedSampler _defaultSampler = new(0.01); // %1

    public override SamplingResult ShouldSample(in SamplingParameters parameters)
    {
        // Hatalı istekler her zaman sample'la
        if (parameters.Tags?.Any(t => t.Key == "error" && (bool)t.Value!) == true)
            return new SamplingResult(SamplingDecision.RecordAndSample);

        // Yavaş istekler her zaman sample'la (parent span'dan bilgi)
        if (parameters.Tags?.Any(t => t.Key == "slow_request") == true)
            return new SamplingResult(SamplingDecision.RecordAndSample);

        // Diğerleri %1 oranında
        return _defaultSampler.ShouldSample(parameters);
    }
}

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetSampler(new ParentBasedSampler(new SmartSampler())));
```

---

## 2. Health Check ve Metrics Endpoint'lerini Loglamak

❌ **Yanlış Kullanım:** Altyapı endpoint'lerini loglara dahil etmek.

```csharp
app.UseSerilogRequestLogging();
// /health, /metrics, /favicon.ico gibi istekler de loglanır
// Günde on binlerce gereksiz log satırı
```

✅ **İdeal Kullanım:** Altyapı endpoint'lerini filtreleyin.

```csharp
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "{RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0}ms";

    options.GetLevel = (context, elapsed, ex) =>
    {
        if (ex != null) return LogEventLevel.Error;
        if (elapsed > 1000) return LogEventLevel.Warning;
        if (context.Response.StatusCode >= 500) return LogEventLevel.Error;
        return LogEventLevel.Information;
    };

    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
        diagnosticContext.Set("ClientIp", httpContext.Connection.RemoteIpAddress);
    };

    // Altyapı endpoint'lerini hariç tut
    options.ShouldLog = (context) =>
    {
        var path = context.Request.Path.Value;
        return !path!.StartsWith("/health") &&
               !path.StartsWith("/metrics") &&
               !path.StartsWith("/favicon");
    };
});
```

---

## 3. Log Enrichment Yapmamak

❌ **Yanlış Kullanım:** Bağlamsız loglar yazmak.

```csharp
_logger.LogError("Sipariş oluşturulamadı");
// Hangi kullanıcı? Hangi sipariş? Hangi ortam? Bilinmiyor
```

✅ **İdeal Kullanım:** Her log satırına bağlam ekleyin.

```csharp
builder.Host.UseSerilog((context, config) => config
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .Enrich.WithProperty("Application", "OrderService")
    .Enrich.WithProperty("Version", typeof(Program).Assembly.GetName().Version?.ToString()));

// Request bazlı enrichment
public class RequestContextMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var userId = context.User.FindFirst("sub")?.Value ?? "anonymous";
        var correlationId = context.Request.Headers["X-Correlation-Id"].FirstOrDefault()
            ?? Activity.Current?.Id ?? Guid.NewGuid().ToString();

        using (LogContext.PushProperty("UserId", userId))
        using (LogContext.PushProperty("CorrelationId", correlationId))
        {
            context.Response.Headers["X-Correlation-Id"] = correlationId;
            await next(context);
        }
    }
}
```

---

## 4. Dynamic Log Level Kullanmamak

❌ **Yanlış Kullanım:** Log seviyesini değiştirmek için yeniden deploy gerekmesi.

```json
{
  "Logging": {
    "LogLevel": { "Default": "Warning" }
  }
}
// Debug bilgisi lazım → deploy gerekir → sorun geçer
```

✅ **İdeal Kullanım:** Runtime'da log seviyesini değiştirin.

```csharp
builder.Host.UseSerilog((context, config) => config
    .ReadFrom.Configuration(context.Configuration)
    .MinimumLevel.ControlledBy(new LoggingLevelSwitch(LogEventLevel.Information)));

// API ile log seviyesi değiştirme
app.MapPost("/admin/log-level", (string level, LoggingLevelSwitch levelSwitch) =>
{
    if (Enum.TryParse<LogEventLevel>(level, true, out var newLevel))
    {
        levelSwitch.MinimumLevel = newLevel;
        return TypedResults.Ok($"Log seviyesi değiştirildi: {newLevel}");
    }
    return TypedResults.BadRequest("Geçersiz seviye");
}).RequireAuthorization("Admin");

// Veya Serilog.Settings.Configuration ile hot-reload
builder.Configuration.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
```

---

## 5. Sensitive Data Filtreleme Yapmamak

❌ **Yanlış Kullanım:** Request/response body'lerini olduğu gibi loglamak.

```csharp
_logger.LogInformation("Request: {@Body}", requestBody);
// Şifre, kredi kartı, kişisel veri loglara yazılır
```

✅ **İdeal Kullanım:** Hassas verileri otomatik maskeleyin.

```csharp
public class SensitiveDataFilter : IDestructuringPolicy
{
    private static readonly HashSet<string> SensitiveKeys = new(StringComparer.OrdinalIgnoreCase)
    {
        "password", "token", "secret", "authorization",
        "cardNumber", "cvv", "ssn", "creditCard"
    };

    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory,
        out LogEventPropertyValue? result)
    {
        if (value is IDictionary<string, object> dict)
        {
            var sanitized = dict.ToDictionary(
                kvp => kvp.Key,
                kvp => SensitiveKeys.Contains(kvp.Key)
                    ? (object)"***REDACTED***"
                    : kvp.Value);

            result = factory.CreatePropertyValue(sanitized, destructureObjects: true);
            return true;
        }

        result = null;
        return false;
    }
}

builder.Host.UseSerilog((context, config) => config
    .Destructure.With<SensitiveDataFilter>());
```
