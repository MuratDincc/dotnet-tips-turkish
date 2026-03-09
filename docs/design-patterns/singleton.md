# Singleton Pattern

Singleton deseni, bir sınıftan yalnızca bir örnek oluşturulmasını garanti eder; ancak yanlış uygulamalar thread-safety sorunlarına ve test edilemez koda yol açabilir.

---

## 1. Thread-Safe Olmayan Singleton

❌ **Yanlış Kullanım:** Thread-safety gözetmeden singleton oluşturmak.

```csharp
public class DatabaseConnection
{
    private static DatabaseConnection _instance;

    public static DatabaseConnection Instance
    {
        get
        {
            if (_instance == null)
            {
                _instance = new DatabaseConnection(); // Race condition riski
            }
            return _instance;
        }
    }
}
```

✅ **İdeal Kullanım:** `Lazy<T>` ile thread-safe singleton oluşturun.

```csharp
public class DatabaseConnection
{
    private static readonly Lazy<DatabaseConnection> _instance =
        new Lazy<DatabaseConnection>(() => new DatabaseConnection());

    public static DatabaseConnection Instance => _instance.Value;

    private DatabaseConnection() { }
}
```

---

## 2. DI Container Yerine Manuel Singleton Yönetimi

❌ **Yanlış Kullanım:** Singleton'ı manuel olarak yönetmek.

```csharp
public class OrderService
{
    private readonly CacheManager _cache = CacheManager.Instance;

    public Order GetOrder(int id)
    {
        return _cache.Get<Order>(id);
    }
}
```

✅ **İdeal Kullanım:** DI container ile singleton lifecycle yönetimi yapın.

```csharp
builder.Services.AddSingleton<ICacheManager, CacheManager>();

public class OrderService
{
    private readonly ICacheManager _cache;

    public OrderService(ICacheManager cache)
    {
        _cache = cache;
    }

    public Order GetOrder(int id)
    {
        return _cache.Get<Order>(id);
    }
}
```

---

## 3. Singleton İçinde Scoped Servisleri Kullanmak

❌ **Yanlış Kullanım:** Singleton servis içinde scoped servisi doğrudan inject etmek.

```csharp
builder.Services.AddSingleton<INotificationService, NotificationService>();

public class NotificationService : INotificationService
{
    private readonly AppDbContext _context; // Scoped servis, captive dependency!

    public NotificationService(AppDbContext context)
    {
        _context = context;
    }
}
```

✅ **İdeal Kullanım:** `IServiceScopeFactory` ile gerektiğinde scope oluşturun.

```csharp
public class NotificationService : INotificationService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public NotificationService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task SendAsync(string message)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // context güvenli şekilde kullanılabilir
    }
}
```

---

## 4. Singleton State ile Test Edilemez Kod

❌ **Yanlış Kullanım:** Statik singleton erişimi ile birim testleri zorlaştırmak.

```csharp
public class ReportService
{
    public Report Generate()
    {
        var config = AppConfig.Instance; // Statik bağımlılık, mock yapılamaz
        return new Report(config.ReportTitle);
    }
}
```

✅ **İdeal Kullanım:** Interface üzerinden bağımlılık alarak test edilebilir hale getirin.

```csharp
public class ReportService
{
    private readonly IAppConfig _config;

    public ReportService(IAppConfig config)
    {
        _config = config;
    }

    public Report Generate()
    {
        return new Report(_config.ReportTitle);
    }
}
```

---

## 5. Singleton ile Dispose Edilemeyen Kaynaklar

❌ **Yanlış Kullanım:** Singleton içinde yönetilmeyen kaynakları temizlememek.

```csharp
public class FileLogger
{
    private static readonly FileLogger _instance = new FileLogger();
    private readonly StreamWriter _writer;

    private FileLogger()
    {
        _writer = new StreamWriter("app.log", append: true);
    }

    public static FileLogger Instance => _instance;

    public void Log(string message) => _writer.WriteLine(message);
    // StreamWriter hiçbir zaman dispose edilmiyor
}
```

✅ **İdeal Kullanım:** `IDisposable` uygulayarak kaynakları düzgün yönetin.

```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _writer;

    public FileLogger()
    {
        _writer = new StreamWriter("app.log", append: true);
    }

    public void Log(string message) => _writer.WriteLine(message);

    public void Dispose()
    {
        _writer?.Dispose();
    }
}

// DI container dispose işlemini otomatik yönetir
builder.Services.AddSingleton<FileLogger>();
```

---

## 6. Double-Check Locking Hatası

❌ **Yanlış Kullanım:** `volatile` anahtar kelimesi olmadan double-check locking yapmak.

```csharp
public class ConnectionPool
{
    private static ConnectionPool _instance;
    private static readonly object _lock = new object();

    public static ConnectionPool Instance
    {
        get
        {
            if (_instance == null)
            {
                lock (_lock)
                {
                    if (_instance == null)
                        _instance = new ConnectionPool(); // Instruction reordering riski
                }
            }
            return _instance;
        }
    }
}
```

✅ **İdeal Kullanım:** `Lazy<T>` kullanarak karmaşık locking'den kaçının.

```csharp
public class ConnectionPool
{
    private static readonly Lazy<ConnectionPool> _instance =
        new Lazy<ConnectionPool>(() => new ConnectionPool());

    public static ConnectionPool Instance => _instance.Value;

    private ConnectionPool() { }
}
```