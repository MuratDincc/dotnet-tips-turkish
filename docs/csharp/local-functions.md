# Local Functions

Local functions, bir metot içinde tanımlanan ve yalnızca o metot içinde kullanılan küçük işlevlerdir. Bu, kodun okunabilirliğini artırır, gereksiz metot tanımlamalarını önler ve kapsamı daraltarak kodun daha düzenli olmasını sağlar. Ancak, yanlış kullanımlar performans sorunlarına ve karmaşıklığa yol açabilir.

---

## 1. Gereksiz Local Function Kullanımı

❌ **Yanlış Kullanım:** Anlamlı bir gereksinim olmadan local function tanımlamak.

```csharp
void PerformCalculation()
{
    void Helper()
    {
        Console.WriteLine("Hesaplama tamamlandı.");
    }
    Helper();
}
```

✅ **İdeal Kullanım:** Local functions yalnızca kodun anlamını ve okunabilirliğini artırmak için kullanılmalıdır.

```csharp
void PerformCalculation()
{
    void AddNumbers(int x, int y) => Console.WriteLine($"Sonuç: {x + y}");
    AddNumbers(5, 10);
}
```

---

## 2. Parametre Geçişlerini Yanlış Kullanmak

❌ **Yanlış Kullanım:** Gereksiz parametre geçişleri yapmak.

```csharp
void PrintMessage(string message)
{
    void Display(string msg) => Console.WriteLine(msg);
    Display(message);
}
```

✅ **İdeal Kullanım:** Gereksiz parametre geçişlerinden kaçının.

```csharp
void PrintMessage(string message)
{
    void Display() => Console.WriteLine(message);
    Display();
}
```

---

## 3. Local Function'ı Yeniden Kullanılamaz Hale Getirmek

❌ **Yanlış Kullanım:** Local function'ı yeniden kullanılabilir kod parçacıkları için kullanmak.

```csharp
void ProcessData(int data)
{
    void MultiplyByTwo(int value) => Console.WriteLine(value * 2);
    MultiplyByTwo(data);
}
```

✅ **İdeal Kullanım:** Yeniden kullanılabilir kodlar için metotlar yerine local function kullanımı sınırlıdır.

```csharp
void MultiplyByTwo(int value) => Console.WriteLine(value * 2);
MultiplyByTwo(5);
MultiplyByTwo(10);
```

---

## 4. Hataları Yönetmemek

❌ **Yanlış Kullanım:** Local function'larda hataları göz ardı etmek.

```csharp
void PerformDivision(int x, int y)
{
    int Divide() => x / y; // Hata riski
    Console.WriteLine(Divide());
}
```

✅ **İdeal Kullanım:** Local function içinde hata yönetimini sağlayın.

```csharp
void PerformDivision(int x, int y)
{
    int Divide()
    {
        if (y == 0) throw new DivideByZeroException("Sıfıra bölme hatası.");
        return x / y;
    }

    try
    {
        Console.WriteLine(Divide());
    }
    catch (DivideByZeroException ex)
    {
        Console.WriteLine($"Hata: {ex.Message}");
    }
}
```

---

## 5. İsimlendirme Standartlarını İhmal Etmek

❌ **Yanlış Kullanım:** Anlamsız ve kısa isimler kullanmak.

```csharp
void DoTask()
{
    void Fn() => Console.WriteLine("Görev tamamlandı.");
    Fn();
}
```

✅ **İdeal Kullanım:** Local function'lar için anlamlı isimler kullanın.

```csharp
void DoTask()
{
    void PrintCompletionMessage() => Console.WriteLine("Görev tamamlandı.");
    PrintCompletionMessage();
}
```

---

## 6. Gereksiz Bağımlılık Eklemek

❌ **Yanlış Kullanım:** Local function'larda gereksiz bağımlılıklar tanımlamak.

```csharp
void ProcessData(int value)
{
    string message = "Sonuç";
    void DisplayResult(int result) => Console.WriteLine($"{message}: {result}");
    DisplayResult(value * 2);
}
```

✅ **İdeal Kullanım:** Bağımlılıkları minimize edin.

```csharp
void ProcessData(int value)
{
    void DisplayResult(int result) => Console.WriteLine($"Sonuç: {result}");
    DisplayResult(value * 2);
}
```