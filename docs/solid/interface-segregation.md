# Interface Segregation Principle (ISP)

İstemciler kullanmadıkları metodlara bağımlı olmamalıdır; şişkin interface'ler gereksiz bağımlılıklara ve boş implementasyonlara yol açar.

---

## 1. Şişkin Interface (Fat Interface)

❌ **Yanlış Kullanım:** Tüm CRUD işlemlerini tek interface'e toplamak.

```csharp
public interface IUserRepository
{
    Task<User> GetByIdAsync(int id);
    Task<IEnumerable<User>> GetAllAsync();
    Task AddAsync(User user);
    Task UpdateAsync(User user);
    Task DeleteAsync(int id);
    Task<User> GetByEmailAsync(string email);
    Task<IEnumerable<User>> SearchAsync(string query);
    Task BulkInsertAsync(IEnumerable<User> users);
    Task ExportToCsvAsync(string path);
}
```

✅ **İdeal Kullanım:** Sorumluluğa göre interface'leri ayırın.

```csharp
public interface IUserReader
{
    Task<User> GetByIdAsync(int id);
    Task<User> GetByEmailAsync(string email);
    Task<IEnumerable<User>> SearchAsync(string query);
}

public interface IUserWriter
{
    Task AddAsync(User user);
    Task UpdateAsync(User user);
    Task DeleteAsync(int id);
}

public interface IUserBulkOperations
{
    Task BulkInsertAsync(IEnumerable<User> users);
    Task ExportToCsvAsync(string path);
}
```

---

## 2. Boş Implementasyonlara Zorlama

❌ **Yanlış Kullanım:** Kullanılmayan metodları boş bırakmak zorunda kalmak.

```csharp
public interface IMessageBroker
{
    Task PublishAsync(string topic, object message);
    Task SubscribeAsync(string topic, Action<object> handler);
    Task AcknowledgeAsync(string messageId);
    Task RejectAsync(string messageId);
}

public class FireAndForgetPublisher : IMessageBroker
{
    public Task PublishAsync(string topic, object message) { /* ... */ }
    public Task SubscribeAsync(string topic, Action<object> handler) => Task.CompletedTask; // Kullanılmıyor
    public Task AcknowledgeAsync(string messageId) => Task.CompletedTask; // Kullanılmıyor
    public Task RejectAsync(string messageId) => Task.CompletedTask; // Kullanılmıyor
}
```

✅ **İdeal Kullanım:** Publisher ve Subscriber interface'lerini ayırın.

```csharp
public interface IMessagePublisher
{
    Task PublishAsync(string topic, object message);
}

public interface IMessageSubscriber
{
    Task SubscribeAsync(string topic, Action<object> handler);
    Task AcknowledgeAsync(string messageId);
    Task RejectAsync(string messageId);
}

public class FireAndForgetPublisher : IMessagePublisher
{
    public Task PublishAsync(string topic, object message) { /* ... */ }
}
```

---

## 3. Servis Interface'inde Rol Karışımı

❌ **Yanlış Kullanım:** Admin ve kullanıcı işlemlerini aynı interface'de toplamak.

```csharp
public interface IProductService
{
    Task<Product> GetByIdAsync(int id);
    Task<IEnumerable<Product>> SearchAsync(string query);
    Task CreateAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
    Task ApproveAsync(int id);
    Task RejectAsync(int id);
    Task BulkDeleteAsync(IEnumerable<int> ids);
}
```

✅ **İdeal Kullanım:** Rol bazlı interface'ler tanımlayın.

```csharp
public interface IProductQueryService
{
    Task<Product> GetByIdAsync(int id);
    Task<IEnumerable<Product>> SearchAsync(string query);
}

public interface IProductManagementService
{
    Task CreateAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
}

public interface IProductAdminService
{
    Task ApproveAsync(int id);
    Task RejectAsync(int id);
    Task BulkDeleteAsync(IEnumerable<int> ids);
}
```

---

## 4. Generic Interface'de Gereksiz Metod

❌ **Yanlış Kullanım:** Her entity türü için geçerli olmayan metodları generic interface'e koymak.

```csharp
public interface IEntity
{
    int Id { get; set; }
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
    string CreatedBy { get; set; }
    bool IsDeleted { get; set; }
    void SoftDelete();
    void Restore();
}

// Audit log entity'sinin soft delete'e ihtiyacı yok
public class AuditLog : IEntity
{
    public void SoftDelete() => throw new NotSupportedException();
    public void Restore() => throw new NotSupportedException();
    // ...
}
```

✅ **İdeal Kullanım:** İlgili davranışları ayrı interface'lere ayırın.

```csharp
public interface IEntity
{
    int Id { get; set; }
}

public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
}

public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    void SoftDelete();
    void Restore();
}

public class Order : IEntity, IAuditable, ISoftDeletable { /* ... */ }
public class AuditLog : IEntity, IAuditable { /* ... */ }
```

---

## 5. Marker Interface Yerine Attribute Kullanmamak

❌ **Yanlış Kullanım:** Boş marker interface'ler ile gereksiz bağımlılık oluşturmak.

```csharp
public interface ICacheable { }
public interface ILoggable { }
public interface IAuditable { }

public class ProductService : IProductService, ICacheable, ILoggable, IAuditable
{
    // ICacheable, ILoggable, IAuditable hiçbir metod tanımlamıyor
}
```

✅ **İdeal Kullanım:** Metadata için attribute kullanın.

```csharp
[Cacheable(Duration = 300)]
[Auditable]
public class ProductService : IProductService
{
    // ...
}

[AttributeUsage(AttributeTargets.Class)]
public class CacheableAttribute : Attribute
{
    public int Duration { get; set; }
}
```