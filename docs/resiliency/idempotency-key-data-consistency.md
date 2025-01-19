# Idempotency Key ve Veri Tutarlılığı

Idempotency, bir işlemin birden fazla kez çağrıldığında aynı sonucu garanti etmesi anlamına gelir. Bu özellik, dağıtık sistemlerde veri tutarlılığını sağlamak ve yinelenen işlemleri önlemek için kritik bir stratejidir.

---

## 1. Idempotency Nedir?

- **Amaç:** Aynı işlem birden fazla kez tekrarlandığında sistemin durumu değiştirmeden aynı sonucu döndürmesi.
- **Kullanım Alanları:** API çağrıları, ödeme sistemleri, mesajlaşma.

Örneğin, bir ödeme işlemi yinelenen bir API çağrısı nedeniyle birden fazla kez işlenmemelidir.

---

## 2. Idempotency Key Kullanımı

Idempotency Key, bir isteği benzersiz şekilde tanımlayan bir anahtardır. Bu anahtar, istemciden gelen isteğin tekrar işlenmesini engeller.

❌ **Yanlış Kullanım:** Idempotency Key olmadan istek işleme

```csharp
public async Task ProcessPaymentAsync(PaymentRequest request)
{
    // Tekrar eden istekleri kontrol etmeden işleme
    await SavePaymentToDatabaseAsync(request);
    await ChargeCustomerAsync(request);
}
```

✅ **İdeal Kullanım:** Idempotency Key ile kontrol

```csharp
public async Task ProcessPaymentAsync(PaymentRequest request)
{
    if (await IsRequestAlreadyProcessedAsync(request.IdempotencyKey))
    {
        Console.WriteLine("Request already processed. Skipping.");
        return;
    }

    await SavePaymentToDatabaseAsync(request);
    await ChargeCustomerAsync(request);
    await MarkRequestAsProcessedAsync(request.IdempotencyKey);
}
```

---

## 3. Idempotency Key ve Veritabanı Kullanımı

Idempotency Key genellikle bir veritabanında saklanır ve her isteğin benzersiz olduğunu garanti eder.

✅ **Örnek:** Veritabanı ile Idempotency Key kontrolü

```csharp
public async Task<bool> IsRequestAlreadyProcessedAsync(string idempotencyKey)
{
    return await _dbContext.IdempotencyKeys.AnyAsync(k => k.Key == idempotencyKey);
}

public async Task MarkRequestAsProcessedAsync(string idempotencyKey)
{
    await _dbContext.IdempotencyKeys.AddAsync(new IdempotencyKey { Key = idempotencyKey });
    await _dbContext.SaveChangesAsync();
}
```

---

## 4. Mesajlaşma Sistemlerinde Idempotency

Mesajlar birden fazla kez işlenebilir, bu yüzden idempotent bir tasarım kullanmak önemlidir.

✅ **Örnek:** Mesaj işleme

```csharp
public async Task ProcessMessageAsync(Message message)
{
    if (await IsMessageAlreadyProcessedAsync(message.Id))
    {
        Console.WriteLine("Message already processed.");
        return;
    }

    await HandleMessageAsync(message);
    await MarkMessageAsProcessedAsync(message.Id);
}
```

---

## 5. Idempotency ve API Tasarımı

REST API'lerde idempotency, istemcilerin güvenilir bir şekilde veri gönderip almasını sağlar.

✅ **Örnek:** Idempotency Key ile API Çağrısı

```http
POST /payments HTTP/1.1
Content-Type: application/json
Idempotency-Key: 12345

{
    "amount": 100,
    "currency": "USD"
}
```

Sunucu tarafında:

```csharp
if (await IsRequestAlreadyProcessedAsync(request.Headers["Idempotency-Key"]))
{
    return Ok("Request already processed.");
}
```

---

## 6. Loglama ve İzleme

Idempotency sistemlerinin düzgün çalıştığından emin olmak için loglama ve izleme araçları kullanın.

✅ **Örnek:** Loglama ile idempotency izleme

```csharp
public async Task ProcessPaymentAsync(PaymentRequest request)
{
    if (await IsRequestAlreadyProcessedAsync(request.IdempotencyKey))
    {
        Console.WriteLine($"Duplicate request detected: {request.IdempotencyKey}");
        return;
    }

    await SavePaymentToDatabaseAsync(request);
    Console.WriteLine($"Processed request: {request.IdempotencyKey}");
    await MarkRequestAsProcessedAsync(request.IdempotencyKey);
}
```