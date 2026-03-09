# Service Discovery

Service Discovery, microservice'lerin birbirini dinamik olarak bulmasını sağlar; hardcoded adresler ortam değişikliklerinde kırılganlığa yol açar.

---

## 1. Hardcoded Servis Adresleri

❌ **Yanlış Kullanım:** Servis adreslerini kod içinde sabitlemek.

```csharp
public class OrderServiceClient
{
    private readonly HttpClient _client;

    public OrderServiceClient(HttpClient client)
    {
        _client = client;
        _client.BaseAddress = new Uri("https://192.168.1.50:5001"); // IP değişirse kod güncellenmeli
    }
}
```

✅ **İdeal Kullanım:** Konfigürasyondan dinamik adres yönetimi yapın.

```csharp
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>((sp, client) =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    client.BaseAddress = new Uri(config["Services:OrderService:BaseUrl"]);
});

// appsettings.json
{
    "Services": {
        "OrderService": {
            "BaseUrl": "https://order-service:5001"
        }
    }
}
```

---

## 2. DNS-Based Discovery Kullanmamak

❌ **Yanlış Kullanım:** Her ortam için ayrı konfigürasyon dosyası yönetmek.

```csharp
// appsettings.Development.json
{ "OrderServiceUrl": "https://localhost:5001" }
// appsettings.Staging.json
{ "OrderServiceUrl": "https://order-staging:5001" }
// appsettings.Production.json
{ "OrderServiceUrl": "https://order-prod:5001" }
```

✅ **İdeal Kullanım:** Kubernetes DNS ile otomatik service discovery yapın.

```csharp
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>(client =>
{
    // Kubernetes'te servis adı DNS üzerinden çözümlenir
    client.BaseAddress = new Uri("http://order-service.default.svc.cluster.local");
});

// Veya kısaca (aynı namespace'de)
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>(client =>
{
    client.BaseAddress = new Uri("http://order-service");
});
```

---

## 3. Consul ile Service Registry

❌ **Yanlış Kullanım:** Servisleri manuel olarak konfigürasyona eklemek.

```csharp
var services = new Dictionary<string, string>
{
    ["order-service"] = "https://order:5001",
    ["payment-service"] = "https://payment:5002",
    // Yeni servis eklemek için deploy gerekir
};
```

✅ **İdeal Kullanım:** Consul ile dinamik servis kayıt ve keşif yapın.

```csharp
builder.Services.AddSingleton<IConsulClient>(sp =>
    new ConsulClient(config => config.Address = new Uri("http://consul:8500")));

// Servis kaydı
public class ConsulRegistrationService : IHostedService
{
    private readonly IConsulClient _consul;
    private string _registrationId;

    public async Task StartAsync(CancellationToken ct)
    {
        _registrationId = $"order-service-{Guid.NewGuid()}";
        await _consul.Agent.ServiceRegister(new AgentServiceRegistration
        {
            ID = _registrationId,
            Name = "order-service",
            Address = "order-service",
            Port = 5001,
            Check = new AgentServiceCheck
            {
                HTTP = "http://order-service:5001/health",
                Interval = TimeSpan.FromSeconds(10)
            }
        }, ct);
    }

    public async Task StopAsync(CancellationToken ct)
    {
        await _consul.Agent.ServiceDeregister(_registrationId, ct);
    }
}
```

---

## 4. Load Balancing Yapmamak

❌ **Yanlış Kullanım:** Tek instance'a sabit bağlantı kurmak.

```csharp
builder.Services.AddHttpClient<IProductService, ProductServiceClient>(client =>
{
    client.BaseAddress = new Uri("https://product-service-1:5001"); // Tek instance
});
```

✅ **İdeal Kullanım:** Client-side load balancing ile birden fazla instance'a dağıtım yapın.

```csharp
// YARP ile load balancing
{
    "Clusters": {
        "product-cluster": {
            "LoadBalancingPolicy": "RoundRobin",
            "Destinations": {
                "instance1": { "Address": "https://product-service-1:5001" },
                "instance2": { "Address": "https://product-service-2:5001" },
                "instance3": { "Address": "https://product-service-3:5001" }
            },
            "HealthCheck": {
                "Active": {
                    "Enabled": true,
                    "Interval": "00:00:10",
                    "Path": "/health"
                }
            }
        }
    }
}
```

---

## 5. Service Discovery Cache Yönetimi

❌ **Yanlış Kullanım:** Her istekte service registry'ye sorgu yapmak.

```csharp
public async Task<string> GetServiceUrlAsync(string serviceName)
{
    var services = await _consul.Health.Service(serviceName); // Her çağrıda Consul'a istek
    return services.Response.First().Service.Address;
}
```

✅ **İdeal Kullanım:** Servis adreslerini cache'leyip periyodik olarak yenileyin.

```csharp
public class ServiceDiscoveryCache : IHostedService
{
    private readonly IConsulClient _consul;
    private readonly ConcurrentDictionary<string, List<string>> _cache = new();
    private Timer _timer;

    public Task StartAsync(CancellationToken ct)
    {
        _timer = new Timer(RefreshCache, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
        return Task.CompletedTask;
    }

    private async void RefreshCache(object state)
    {
        var services = await _consul.Health.Service("order-service");
        var addresses = services.Response
            .Where(s => s.Checks.All(c => c.Status == HealthStatus.Passing))
            .Select(s => $"https://{s.Service.Address}:{s.Service.Port}")
            .ToList();

        _cache["order-service"] = addresses;
    }

    public List<string> GetAddresses(string serviceName)
        => _cache.GetValueOrDefault(serviceName, new List<string>());

    public Task StopAsync(CancellationToken ct) { _timer?.Dispose(); return Task.CompletedTask; }
}
```