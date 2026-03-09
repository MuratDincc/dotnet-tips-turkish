# Span<T> ve Memory Optimizasyonu

Span<T> ve Memory<T>, heap allocation olmadan bellek üzerinde verimli işlem yapmayı sağlar; yanlış kullanımlar gereksiz bellek tüketimine yol açar.

---

## 1. Substring ile Gereksiz Allocation

❌ **Yanlış Kullanım:** String.Substring ile yeni string nesneleri oluşturmak.

```csharp
public string GetDomain(string email)
{
    var atIndex = email.IndexOf('@');
    return email.Substring(atIndex + 1); // Yeni string nesnesi oluşur
}
```

✅ **İdeal Kullanım:** Span ile allocation-free string işlemi yapın.

```csharp
public ReadOnlySpan<char> GetDomain(ReadOnlySpan<char> email)
{
    var atIndex = email.IndexOf('@');
    return email[(atIndex + 1)..]; // Yeni nesne oluşmaz, mevcut belleğe referans
}
```

---

## 2. Array Kopyalama ile Allocation

❌ **Yanlış Kullanım:** Array.Copy ile yeni array oluşturmak.

```csharp
public byte[] GetHeader(byte[] data)
{
    var header = new byte[4];
    Array.Copy(data, 0, header, 0, 4); // Yeni array allocation
    return header;
}
```

✅ **İdeal Kullanım:** Span slice ile kopyalama olmadan erişin.

```csharp
public ReadOnlySpan<byte> GetHeader(ReadOnlySpan<byte> data)
{
    return data[..4]; // Allocation yok
}
```

---

## 3. String Concatenation ile Bellek İsrafı

❌ **Yanlış Kullanım:** Döngüde string birleştirme yapmak.

```csharp
public string BuildCsv(IEnumerable<Product> products)
{
    var result = "";
    foreach (var product in products)
    {
        result += $"{product.Id},{product.Name},{product.Price}\n"; // Her iterasyonda yeni string
    }
    return result;
}
```

✅ **İdeal Kullanım:** StringBuilder veya string.Create kullanın.

```csharp
public string BuildCsv(IEnumerable<Product> products)
{
    var sb = new StringBuilder(1024);
    foreach (var product in products)
    {
        sb.Append(product.Id).Append(',')
          .Append(product.Name).Append(',')
          .Append(product.Price).AppendLine();
    }
    return sb.ToString();
}
```

---

## 4. ArrayPool Kullanmamak

❌ **Yanlış Kullanım:** Her işlemde yeni byte array oluşturmak.

```csharp
public async Task ProcessFileAsync(Stream stream)
{
    var buffer = new byte[8192]; // Her çağrıda heap allocation
    int bytesRead;
    while ((bytesRead = await stream.ReadAsync(buffer)) > 0)
    {
        ProcessChunk(buffer, bytesRead);
    }
}
```

✅ **İdeal Kullanım:** ArrayPool ile buffer'ları yeniden kullanın.

```csharp
public async Task ProcessFileAsync(Stream stream)
{
    var buffer = ArrayPool<byte>.Shared.Rent(8192);
    try
    {
        int bytesRead;
        while ((bytesRead = await stream.ReadAsync(buffer)) > 0)
        {
            ProcessChunk(buffer.AsSpan(0, bytesRead));
        }
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

---

## 5. stackalloc Kullanmamak

❌ **Yanlış Kullanım:** Küçük geçici bufferlar için heap allocation.

```csharp
public string FormatId(int id)
{
    var bytes = new byte[16]; // Heap allocation
    // Format işlemleri
    return Convert.ToHexString(bytes);
}
```

✅ **İdeal Kullanım:** Küçük bufferlar için stackalloc ile stack üzerinde yer ayırın.

```csharp
public string FormatId(int id)
{
    Span<byte> bytes = stackalloc byte[16]; // Stack allocation, GC yükü yok
    BitConverter.TryWriteBytes(bytes, id);
    return Convert.ToHexString(bytes);
}

public bool TryParseDate(ReadOnlySpan<char> input, out DateTime result)
{
    Span<char> buffer = stackalloc char[10];
    // Stack üzerinde geçici çalışma alanı
    return DateTime.TryParseExact(input, "yyyy-MM-dd", null,
        System.Globalization.DateTimeStyles.None, out result);
}
```