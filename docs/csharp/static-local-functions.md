# Static Local Functions

Static local functions, C# dilinde performans ve güvenlik avantajları sağlamak için kullanılabilir. Bu metotlar, dış kapsamdaki hiçbir değişkene erişemez ve bu nedenle bellek kullanımını optimize eder. Ancak, yanlış kullanımları kodun performansını düşürebilir ve anlaşılabilirliğini zorlaştırabilir.

---

## 1. Gereksiz Static Local Function Kullanımı

❌ **Yanlış Kullanım:** Static local functions'ı gereksiz durumlarda kullanmak.

```csharp
void CalculateSum(int a, int b)
{
    static int Add(int x, int y) => x + y; // Gereksiz static kullanımı
    Console.WriteLine(Add(a, b));
}
```

✅ **İdeal Kullanım:** Static local functions'ı yalnızca dış kapsama erişim gerekmediğinde kullanın.

```csharp
void CalculateSum(int a, int b)
{
    int Add(int x, int y) => x + y;
    Console.WriteLine(Add(a, b));
}
```

---

## 2. Dış Değişkenlere Erişmeye Çalışmak

❌ **Yanlış Kullanım:** Static local function içinde dış değişkenlere erişmek.

```csharp
int multiplier = 2;
void MultiplyAndPrint(int number)
{
    static int Multiply(int x) => x * multiplier; // Hata: Static metot dış değişkenlere erişemez
    Console.WriteLine(Multiply(number));
}
```

✅ **İdeal Kullanım:** Gerekli veriyi parametre olarak iletin.

```csharp
int multiplier = 2;
void MultiplyAndPrint(int number)
{
    static int Multiply(int x, int factor) => x * factor;
    Console.WriteLine(Multiply(number, multiplier));
}
```

---

## 3. Anlamsız İsimlendirme

❌ **Yanlış Kullanım:** Static local function için anlamsız ve kısa isimler kullanmak.

```csharp
void ProcessData(int data)
{
    static int Fn(int x) => x * 2;
    Console.WriteLine(Fn(data));
}
```

✅ **İdeal Kullanım:** Metot isimlerini açıklayıcı ve anlamlı seçin.

```csharp
void ProcessData(int data)
{
    static int DoubleValue(int x) => x * 2;
    Console.WriteLine(DoubleValue(data));
}
```

---

## 4. Performans İyileştirmelerini Göz Ardı Etmek

❌ **Yanlış Kullanım:** Performans avantajı sağlamayacak durumda static local function kullanmak.

```csharp
void PrintNumbers()
{
    static void Print(int x) => Console.WriteLine(x);
    for (int i = 0; i < 5; i++)
    {
        Print(i); // Performans farkı yaratmaz
    }
}
```

✅ **İdeal Kullanım:** Performans kritik durumlarda static local function tercih edin.

```csharp
void ProcessLargeData(int[] numbers)
{
    static int Process(int x) => x * 2;
    foreach (var number in numbers)
    {
        Console.WriteLine(Process(number));
    }
}
```

---

## 5. Gereksiz Bağımlılıklar Eklemek

❌ **Yanlış Kullanım:** Static local function'da dış metotlara gereksiz bağımlılık eklemek.

```csharp
void ProcessNumbers()
{
    static int AddAndDouble(int x, int y)
    {
        return (x + y) * 2;
    }
    Console.WriteLine(AddAndDouble(3, 5));
}
```

✅ **İdeal Kullanım:** Gerekli bağımlılıkları minimize edin.

```csharp
void ProcessNumbers()
{
    static int DoubleSum(int x, int y) => (x + y) * 2;
    Console.WriteLine(DoubleSum(3, 5));
}
```