# Dependency Injection (DI)

Dependency Injection (DI), bir uygulamada bağımlılıkların yönetimini kolaylaştırarak test edilebilirliği ve kodun sürdürülebilirliğini artırır. Ancak DI yanlış kullanıldığında kodun karmaşıklığını artırabilir.

---

## 1. Bağımlılıkların Elle Yönetilmesi

❌ **Yanlış Kullanım:** Bağımlılıkları elle örneklemek.

```csharp
public class OrderService
{
    private readonly ProductRepository _productRepository;

    public OrderService()
    {
        _productRepository = new ProductRepository(); // Sıkı bağımlılık
    }
}
```

✅ **İdeal Kullanım:** Bağımlılıkları bir IoC konteynırı üzerinden enjekte edin.

```csharp
public class OrderService
{
    private readonly IProductRepository _productRepository;

    public OrderService(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }
}
```

---

## 2. Yanlış Yaşam Süresi (Lifetime) Yönetimi

❌ **Yanlış Kullanım:** Scoped bağımlılıkları singleton bir hizmete enjekte etmek.

```csharp
services.AddSingleton<MyService>();
services.AddScoped<MyDbContext>();

public class MyService
{
    private readonly MyDbContext _dbContext;

    public MyService(MyDbContext dbContext)
    {
        _dbContext = dbContext; // Hatalı yaşam süresi
    }
}
```

✅ **İdeal Kullanım:** Bağımlılıkların yaşam süresini doğru yapılandırın.

```csharp
services.AddScoped<MyService>();
services.AddScoped<MyDbContext>();

public class MyService
{
    private readonly MyDbContext _dbContext;

    public MyService(MyDbContext dbContext)
    {
        _dbContext = dbContext;
    }
}
```

---

## 3. Çok Fazla Bağımlılık Enjeksiyonu

❌ **Yanlış Kullanım:** Bir sınıfta çok fazla bağımlılığı doğrudan enjekte etmek.

```csharp
public class MyController
{
    public MyController(IService1 service1, IService2 service2, IService3 service3, IService4 service4)
    {
        // Çok fazla bağımlılık
    }
}
```

✅ **İdeal Kullanım:** Bağımlılıkları gruplandırarak bir arayüz ile soyutlayın.

```csharp
public interface IServiceGroup
{
    IService1 Service1 { get; }
    IService2 Service2 { get; }
}

public class ServiceGroup : IServiceGroup
{
    public IService1 Service1 { get; }
    public IService2 Service2 { get; }

    public ServiceGroup(IService1 service1, IService2 service2)
    {
        Service1 = service1;
        Service2 = service2;
    }
}
```

---

## 4. Service Locator Kullanımı

❌ **Yanlış Kullanım:** Service locator anti-pattern’ini kullanmak.

```csharp
public class MyService
{
    public void DoWork()
    {
        var service = ServiceLocator.GetService<IMyDependency>();
        service.PerformAction();
    }
}
```

✅ **İdeal Kullanım:** Bağımlılıkları doğrudan enjekte edin.

```csharp
public class MyService
{
    private readonly IMyDependency _myDependency;

    public MyService(IMyDependency myDependency)
    {
        _myDependency = myDependency;
    }

    public void DoWork()
    {
        _myDependency.PerformAction();
    }
}
```

---

## 5. Test Edilebilirliği Göz Ardı Etmek

❌ **Yanlış Kullanım:** Sıkı bağımlılıklar nedeniyle test edilemeyen sınıflar.

```csharp
public class ReportService
{
    private readonly Logger _logger = new Logger();

    public void GenerateReport()
    {
        _logger.Log("Rapor oluşturuluyor...");
    }
}
```

✅ **İdeal Kullanım:** Logger gibi bağımlılıkları enjekte ederek test edilebilirliği artırın.

```csharp
public class ReportService
{
    private readonly ILogger _logger;

    public ReportService(ILogger logger)
    {
        _logger = logger;
    }

    public void GenerateReport()
    {
        _logger.Log("Rapor oluşturuluyor...");
    }
}
```

---

## 6. Hizmetlerin Aşırı Kaydı

❌ **Yanlış Kullanım:** Her bağımlılığı manuel olarak kaydetmek.

```csharp
services.AddSingleton<IService1, Service1>();
services.AddSingleton<IService2, Service2>();
services.AddSingleton<IService3, Service3>();
```

✅ **İdeal Kullanım:** Assembly tarama ile hizmetleri otomatik olarak kaydedin.

```csharp
var assemblies = AppDomain.CurrentDomain.GetAssemblies();
services.Scan(scan => scan
    .FromAssemblies(assemblies)
    .AddClasses()
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```