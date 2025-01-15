# Async Streams

Async streams, veri akışlarını asenkron olarak işlemek için güçlü bir araçtır. Doğru şekilde kullanıldığında performansı artırır ve kodunuzu daha verimli hale getirir. Ancak, yanlış kullanımlar performans sorunlarına ve beklenmeyen davranışlara yol açabilir.

---

## 1. Asenkron Akışların Yanlış Kullanımı

❌ **Yanlış Kullanım:** Tüm veri setini bellekte tutarak işlem yapmak.

```csharp
var data = await GetDataAsync();
foreach (var item in data)
{
    Console.WriteLine(item);
}
```

✅ **İdeal Kullanım:** Asenkron veri akışını `await foreach` ile tüketmek.

```csharp
await foreach (var item in GetDataAsync())
{
    Console.WriteLine(item);
}
```

---

## 2. Exception Yönetimini İhmal Etmek

❌ **Yanlış Kullanım:** Asenkron akışlar sırasında hataları göz ardı etmek.

```csharp
await foreach (var data in GetDataAsync())
{
    ProcessData(data); // Hataları ele almaz
}
```

✅ **İdeal Kullanım:** Hataları `try-catch` bloğu ile yönetmek.

```csharp
try
{
    await foreach (var data in GetDataAsync())
    {
        ProcessData(data);
    }
}
catch (Exception ex)
{
    Console.WriteLine($"Hata: {ex.Message}");
}
```

---

## 3. Performansı Optimize Etmemek

❌ **Yanlış Kullanım:** Tüm elemanları aynı anda işlemek.

```csharp
await foreach (var item in GetLargeDataAsync())
{
    ProcessItem(item);
}
```

✅ **İdeal Kullanım:** Akışın erken sonlandırılabileceği durumlarda döngüyü zamanında durdurmak.

```csharp
await foreach (var item in GetLargeDataAsync())
{
    if (ShouldStopProcessing(item)) break;
    ProcessItem(item);
}
```

---

## 4. `CancellationToken` Kullanımını İhmal Etmek

❌ **Yanlış Kullanım:** Asenkron akışlarda iptal desteğini göz ardı etmek.

```csharp
await foreach (var item in GetDataAsync())
{
    Console.WriteLine(item);
}
```

✅ **İdeal Kullanım:** `CancellationToken` kullanarak işlem iptalini desteklemek.

```csharp
await foreach (var item in GetDataAsync().WithCancellation(cancellationToken))
{
    Console.WriteLine(item);
}
```

---

## 5. Gereksiz Veri Filtreleme

❌ **Yanlış Kullanım:** Akıştan alınan verileri döngü içinde filtrelemek.

```csharp
await foreach (var item in GetDataAsync())
{
    if (item.IsRelevant)
    {
        Console.WriteLine(item);
    }
}
```

✅ **İdeal Kullanım:** Akış sırasında veri filtrelemeyi optimize etmek.

```csharp
await foreach (var item in GetFilteredDataAsync())
{
    Console.WriteLine(item);
}
```

---

## 6. Bağımsız İşlemleri Senkronize Çalıştırmak

❌ **Yanlış Kullanım:** Bağımsız işlemleri sırayla çalıştırmak.

```csharp
await foreach (var item in GetDataAsync())
{
    await ProcessItemAsync(item); // Bağımsız işlemler sıralı çalışır
}
```

✅ **İdeal Kullanım:** İşlemleri paralel olarak çalıştırmak için `Task.WhenAll` kullanın.

```csharp
var tasks = new List<Task>();
await foreach (var item in GetDataAsync())
{
    tasks.Add(ProcessItemAsync(item));
}
await Task.WhenAll(tasks);
```