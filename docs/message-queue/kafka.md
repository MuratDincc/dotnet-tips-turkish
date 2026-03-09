# Apache Kafka

Kafka, yüksek hacimli event streaming için tasarlanmış dağıtık bir platformdur; yanlış yapılandırma veri kaybına ve consumer lag sorunlarına yol açar.

---

## 1. Partition Stratejisi Belirlememek

❌ **Yanlış Kullanım:** Varsayılan partition ile sıralama garantisi kaybetmek.

```csharp
await producer.ProduceAsync("orders", new Message<Null, string>
{
    Value = JsonSerializer.Serialize(order)
});
// Key yok, round-robin ile partition seçilir
// Aynı müşterinin siparişleri farklı partition'lara dağılır, sıra bozulur
```

✅ **İdeal Kullanım:** Message key ile aynı entity'nin mesajlarını aynı partition'a yönlendirin.

```csharp
await producer.ProduceAsync("orders", new Message<string, string>
{
    Key = order.CustomerId.ToString(), // Aynı müşterinin siparişleri aynı partition'da
    Value = JsonSerializer.Serialize(order)
});
```

---

## 2. Consumer Group Kullanmamak

❌ **Yanlış Kullanım:** Her consumer'ın tüm mesajları okuması.

```csharp
var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    // GroupId yok, her consumer tüm mesajları okur
};
```

✅ **İdeal Kullanım:** Consumer group ile mesajları paralel işleyin.

```csharp
var config = new ConsumerConfig
{
    BootstrapServers = "localhost:9092",
    GroupId = "order-processing-group",
    AutoOffsetReset = AutoOffsetReset.Earliest,
    EnableAutoCommit = false
};

using var consumer = new ConsumerBuilder<string, string>(config).Build();
consumer.Subscribe("orders");

while (!ct.IsCancellationRequested)
{
    var result = consumer.Consume(ct);
    await ProcessAsync(result.Message.Value);
    consumer.Commit(result); // Manuel commit
}
```

---

## 3. Offset Yönetimini Yanlış Yapmak

❌ **Yanlış Kullanım:** Auto commit ile mesaj işlenmeden offset ilerletmek.

```csharp
var config = new ConsumerConfig
{
    EnableAutoCommit = true,       // İşlenmeden commit edilir
    AutoCommitIntervalMs = 5000
};
// Uygulama çökerse son 5 saniyenin mesajları kaybolabilir
```

✅ **İdeal Kullanım:** Manuel commit ile mesaj işlendikten sonra offset kaydedin.

```csharp
var config = new ConsumerConfig
{
    EnableAutoCommit = false
};

while (!ct.IsCancellationRequested)
{
    var result = consumer.Consume(ct);

    try
    {
        await ProcessAsync(result.Message.Value);
        consumer.Commit(result);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Mesaj işlenemedi, offset commit edilmedi");
        // Offset ilerlemez, mesaj tekrar okunur
    }
}
```

---

## 4. Idempotent Producer Kullanmamak

❌ **Yanlış Kullanım:** Network hatalarında duplicate mesaj riski.

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    Acks = Acks.Leader // Sadece leader onayı, replica'lar yazamadan çökerse veri kaybolur
};
```

✅ **İdeal Kullanım:** Idempotent producer ile exactly-once semantics sağlayın.

```csharp
var config = new ProducerConfig
{
    BootstrapServers = "localhost:9092",
    Acks = Acks.All,
    EnableIdempotence = true,
    MaxInFlight = 5,
    MessageSendMaxRetries = 3,
    LingerMs = 5 // Batch gönderim için küçük gecikme
};

using var producer = new ProducerBuilder<string, string>(config).Build();
```

---

## 5. Schema Registry Kullanmamak

❌ **Yanlış Kullanım:** Mesaj formatını kontrol etmemek.

```csharp
// Producer v1 formatında gönderir
producer.Produce("orders", new { OrderId = 1, Total = 100 });

// Consumer v2 formatı bekler
// { OrderId, Total, Currency } - Currency alanı yok, deserialize hatası
```

✅ **İdeal Kullanım:** Schema Registry ile mesaj kontratını yönetin.

```csharp
var schemaRegistryConfig = new SchemaRegistryConfig
{
    Url = "http://localhost:8081"
};

var avroSerializerConfig = new AvroSerializerConfig
{
    AutoRegisterSchemas = true
};

using var schemaRegistry = new CachedSchemaRegistryClient(schemaRegistryConfig);
using var producer = new ProducerBuilder<string, OrderCreated>(
    new ProducerConfig { BootstrapServers = "localhost:9092" })
    .SetValueSerializer(new AvroSerializer<OrderCreated>(schemaRegistry, avroSerializerConfig))
    .Build();

// Schema uyumsuz mesaj gönderilmeye çalışılırsa hata alınır
```