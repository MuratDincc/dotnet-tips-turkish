# Best Practices

MediatR ve CQRS pattern'ının etkili kullanımı için best practice'ler.

## Command ve Query Best Practices

1. **Command'lar**
   - Immutable olmalı
   - Tek sorumluluğu olmalı
   - Validation kuralları tanımlanmalı
   - Idempotent olmalı

2. **Query'ler**
   - Read-only olmalı
   - Performans optimize edilmeli
   - Caching kullanılmalı
   - Pagination desteklemeli

3. **Handler'lar**
   - Tek sorumluluğu olmalı
   - Business logic içermeli
   - Bağımlılıkları minimize etmeli
   - Test edilebilir olmalı

## Pipeline Best Practices

1. **Behavior Sıralaması**
   - Validation
   - Authorization
   - Logging
   - Transaction
   - Caching
   - Performance

2. **Behavior'lar**
   - Hafif olmalı
   - Gereksiz işlemlerden kaçınılmalı
   - Exception handling yapılmalı
   - Test edilebilir olmalı

## Event Best Practices

1. **Event'ler**
   - Immutable olmalı
   - Geçmiş zaman kullanmalı
   - Domain'e özgü olmalı
   - Idempotent olmalı

2. **Event Handler'lar**
   - Tek sorumluluğu olmalı
   - Bağımsız olmalı
   - Retry mekanizması olmalı
   - Test edilebilir olmalı

## Validation Best Practices

1. **Validation Rules**
   - Her request için ayrı validator
   - Kurallar açık ve anlaşılır olmalı
   - Custom validation kullanılmalı
   - Async validation desteklenmeli

2. **Error Messages**
   - Açıklayıcı olmalı
   - Çoklu dil desteği olmalı
   - Hata kodları kullanılmalı
   - Sensitive data içermemeli

## Error Handling Best Practices

1. **Exception Types**
   - Her hata tipi için özel exception
   - Mesajlar açıklayıcı olmalı
   - Inner exception'lar korunmalı
   - Stack trace korunmalı

2. **Error Response**
   - HTTP status code'ları doğru kullanılmalı
   - Response'lar standart olmalı
   - Hata detayları uygun seviyede olmalı
   - Sensitive data içermemeli

## Testing Best Practices

1. **Unit Testing**
   - Her handler için test
   - Tüm senaryolar test edilmeli
   - Mock'lar doğru kullanılmalı
   - Test coverage yüksek olmalı

2. **Integration Testing**
   - Pipeline'lar test edilmeli
   - Behavior'lar test edilmeli
   - Gerçek bağımlılıklar kullanılmalı
   - Performance testleri yapılmalı

## Project Structure Best Practices

1. **Folder Structure**
   ```
   Project/
   ├── Commands/
   │   ├── CreateProduct/
   │   │   ├── CreateProductCommand.cs
   │   │   ├── CreateProductCommandHandler.cs
   │   │   └── CreateProductCommandValidator.cs
   │   └── UpdateProduct/
   ├── Queries/
   │   ├── GetProductById/
   │   │   ├── GetProductByIdQuery.cs
   │   │   └── GetProductByIdQueryHandler.cs
   │   └── GetProducts/
   ├── Events/
   │   ├── ProductCreated/
   │   │   ├── ProductCreatedEvent.cs
   │   │   └── ProductCreatedEventHandler.cs
   │   └── ProductUpdated/
   └── Behaviors/
       ├── LoggingBehavior.cs
       └── ValidationBehavior.cs
   ```

2. **Naming Conventions**
   - Command'lar: `{Action}{Entity}Command`
   - Query'ler: `Get{Entity}{By}{Criteria}Query`
   - Event'ler: `{Entity}{Action}Event`
   - Handler'lar: `{Command/Query/Event}Handler`

## Performance Best Practices

1. **Command Performance**
   - Batch işlemler kullanılmalı
   - Transaction'lar optimize edilmeli
   - Index'ler doğru kullanılmalı
   - Bulk insert/update kullanılmalı

2. **Query Performance**
   - Select sadece ihtiyaç duyulan alanları
   - Include'ları dikkatli kullanın
   - Index'leri doğru kullanın
   - Caching kullanın

## Security Best Practices

1. **Authorization**
   - Role-based authorization
   - Policy-based authorization
   - Resource-based authorization
   - Claims-based authorization

2. **Data Protection**
   - Sensitive data şifrelenmeli
   - Audit logging yapılmalı
   - Input validation yapılmalı
   - Output encoding yapılmalı 