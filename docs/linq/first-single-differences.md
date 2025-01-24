# LINQ: First ve Single Farkları

LINQ'da `First`, `FirstOrDefault`, `Single` ve `SingleOrDefault` metotları, koleksiyonlardan belirli bir eleman seçmek için kullanılır. Ancak bu metotların yanlış kullanımı performans ve hata yönetimi açısından sorunlara yol açabilir.

---

## 1. First ve Single Nedir?

### **First**
- Koleksiyondaki ilk elemanı döndürür.
- Eleman yoksa `InvalidOperationException` fırlatır.
- İlk eleman varsa hemen döner ve işlem sona erer.

**Örnek:**

```csharp
var firstCustomer = customers.First(c => c.IsActive);
Console.WriteLine(firstCustomer.Name);
```

---

### **Single**
- Koleksiyonda yalnızca bir eleman varsa o elemanı döndürür.
- Birden fazla eleman varsa `InvalidOperationException` fırlatır.
- Eleman yoksa yine `InvalidOperationException` fırlatır.

**Örnek:**

```csharp
var singleCustomer = customers.Single(c => c.Id == 1);
Console.WriteLine(singleCustomer.Name);
```

---

## 2. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** First ile benzersiz bir eleman kontrolü

❌ **Yanlış Kullanım:**

```csharp
var singleCustomer = customers.First(c => c.Id == 1);
```

Bu kullanım, koleksiyonun benzersiz bir eleman içerip içermediğini kontrol etmez ve yanlış sonuçlara yol açabilir.

---

### **İdeal Kullanım:** Single ile benzersizlik kontrolü

✅ **İdeal Kullanım:**

```csharp
var singleCustomer = customers.Single(c => c.Id == 1);
```

Bu yöntem, koleksiyonda yalnızca bir eleman olduğundan emin olur.

---

## 3. Default Değer Desteği

Eğer koleksiyonda eleman olmayabileceğini düşünüyorsanız `FirstOrDefault` veya `SingleOrDefault` kullanabilirsiniz.

✅ **Örnek:**

```csharp
var firstCustomer = customers.FirstOrDefault(c => c.IsActive);
if (firstCustomer != null)
{
    Console.WriteLine(firstCustomer.Name);
}
```

```csharp
var singleCustomer = customers.SingleOrDefault(c => c.Id == 1);
if (singleCustomer != null)
{
    Console.WriteLine(singleCustomer.Name);
}
```

---

## 4. Performans Farkları

- **First:** İlk eşleşmeyi bulduğunda işlem sona erer, bu nedenle daha hızlıdır.
- **Single:** Koleksiyonun tamamını tarar, çünkü benzersizlik kontrolü yapar.

✅ **Performans Karşılaştırması:**

```csharp
var stopwatch = Stopwatch.StartNew();

// First kullanımı
var firstCustomer = customers.First(c => c.IsActive);
stopwatch.Stop();
Console.WriteLine($"First Süresi: {stopwatch.ElapsedMilliseconds} ms");

stopwatch.Restart();

// Single kullanımı
var singleCustomer = customers.Single(c => c.Id == 1);
stopwatch.Stop();
Console.WriteLine($"Single Süresi: {stopwatch.ElapsedMilliseconds} ms");
```

---

## 5. Hangi Durumda Hangisi Kullanılmalı?

| **Metot**        | **Kullanım Durumu**                                             |
|-------------------|-----------------------------------------------------------------|
| **First**         | İlk elemanı almak istediğinizde.                                |
| **FirstOrDefault**| İlk eleman yoksa varsayılan bir değer döndürmek istediğinizde.  |
| **Single**        | Koleksiyonda yalnızca bir eleman olduğundan emin olduğunuzda.   |
| **SingleOrDefault**| Koleksiyonda yalnızca bir eleman varsa döndürmek istediğinizde.|