# Null Handling

C# dilinde `null` değerlerin yanlış yönetimi, runtime hatalarına ve beklenmeyen davranışlara yol açabilir. Bu rehber, null handling ile ilgili yaygın hataları ve en iyi uygulamaları içerir.

---

## 1. Null Kontrollerini İhmal Etmek

❌ **Yanlış Kullanım:** `null` kontrolleri yapmamak.

```csharp
public void PrintMessage(string message)
{
    Console.WriteLine(message.Length); // NullReferenceException oluşabilir
}
```

✅ **İdeal Kullanım:** `null` kontrolleri yaparak hataları önleyin.

```csharp
public void PrintMessage(string message)
{
    if (message == null) throw new ArgumentNullException(nameof(message));
    Console.WriteLine(message.Length);
}
```

---

## 2. `null` için Magic Value Kullanımı

❌ **Yanlış Kullanım:** `null` yerine anlamsız bir magic value kullanmak.

```csharp
public string GetMessage() => "";
```

✅ **İdeal Kullanım:** `null` için doğru bir model kullanarak daha okunabilir kod yazın.

```csharp
public string GetMessage() => null;
```

---

## 3. Null Coalescing Operatörünü Kullanmayı Unutmak

❌ **Yanlış Kullanım:** `null` kontrolünü manuel olarak yapmak.

```csharp
var result = value != null ? value : "Default";
```

✅ **İdeal Kullanım:** Null coalescing operatörü (`??`) kullanarak kodu sadeleştirin.

```csharp
var result = value ?? "Default";
```

---

## 4. Null Conditional Operatörünü İhmal Etmek

❌ **Yanlış Kullanım:** Null kontrolü yapmadan zincirleme erişim.

```csharp
var length = person.Address.City.Length; // NullReferenceException riski
```

✅ **İdeal Kullanım:** Null conditional operatörünü (`?.`) kullanarak hataları önleyin.

```csharp
var length = person?.Address?.City?.Length;
```

---

## 5. `Nullable<T>` Kullanımını Göz Ardı Etmek

❌ **Yanlış Kullanım:** Nullable değer türleri ile çalışırken manuel null kontrolü yapmak.

```csharp
int? number = null;
if (number.HasValue) Console.WriteLine(number.Value);
```

✅ **İdeal Kullanım:** `Nullable<T>` özelliklerini kullanarak kodu sadeleştirin.

```csharp
int? number = null;
Console.WriteLine(number ?? 0); // Varsayılan değeri kullanır
```

---

## 6. `null` Döndüren Metotlar Kullanmak

❌ **Yanlış Kullanım:** `null` döndüren metotlar kullanarak hataya açık bir yapı oluşturmak.

```csharp
public string GetData()
{
    return null; // NullReferenceException riski
}
```

✅ **İdeal Kullanım:** Null Object Pattern veya alternatif bir çözüm kullanın.

```csharp
public string GetData()
{
    return string.Empty; // Null yerine boş bir değer döner
}
```

---

## 7. `ArgumentNullException` ile Detay Sağlamamak

❌ **Yanlış Kullanım:** `ArgumentNullException` kullanırken detay sağlamamak.

```csharp
throw new ArgumentNullException();
```

✅ **İdeal Kullanım:** Parametre adı ve açıklama ekleyerek detay sağlayın.

```csharp
throw new ArgumentNullException(nameof(parameter), "Parametre boş olamaz.");
```