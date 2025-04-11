# Temel Kavramlar

## MediatR Nedir?

MediatR, in-process mesajlaşma için basit bir mediator pattern implementasyonudur. Temel özellikleri:

- Request/Response pattern'ı
- Notification pattern'ı
- Pipeline behavior'lar
- Basit ve hafif yapı

### MediatR'ın Avantajları

1. **Loose Coupling**: Sınıflar arası bağımlılıkları azaltır
2. **Single Responsibility**: Her handler tek bir iş yapar
3. **Testability**: Test edilebilirliği artırır
4. **Cross-Cutting Concerns**: Logging, validation gibi işlemleri merkezi olarak yönetir

## CQRS Nedir?

CQRS (Command Query Responsibility Segregation), veri okuma (Query) ve yazma (Command) işlemlerini birbirinden ayıran bir pattern'dır.

### CQRS'in Avantajları

1. **Ölçeklenebilirlik**: Okuma ve yazma işlemleri ayrı ayrı ölçeklendirilebilir
2. **Optimizasyon**: Her işlem için özel optimizasyonlar yapılabilir
3. **Bakım Kolaylığı**: Kod daha organize ve bakımı daha kolay
4. **Performans**: Okuma ve yazma işlemleri için farklı veri modelleri kullanılabilir

## MediatR ve CQRS Birlikte Kullanımı

MediatR ve CQRS birlikte kullanıldığında:

1. **Command'lar**: Veri değiştirme işlemleri için
2. **Query'ler**: Veri okuma işlemleri için
3. **Event'ler**: Domain event'leri için
4. **Behavior'lar**: Cross-cutting concern'ler için

kullanılır.

### Örnek Kullanım

```csharp
// Command örneği
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Query örneği
public class GetProductByIdQuery : IRequest<ProductDto>
{
    public int Id { get; set; }
}

// Event örneği
public class ProductCreatedEvent : INotification
{
    public int ProductId { get; set; }
}
```

## Temel Bileşenler

1. **IRequest<T>**: Command ve Query'ler için temel interface
2. **IRequestHandler<TRequest, TResponse>**: Command ve Query handler'ları için interface
3. **INotification**: Event'ler için interface
4. **INotificationHandler<T>**: Event handler'ları için interface
5. **IPipelineBehavior<TRequest, TResponse>**: Pipeline behavior'lar için interface 