# Pattern Matching

C# dilinde Pattern Matching, daha okunabilir ve sürdürülebilir kod yazmayı sağlayan güçlü bir araçtır. Ancak, yanlış kullanımlar performans sorunlarına ve karmaşık kod yapılarına neden olabilir.

---

## 1. `is` Operatörü ile Gereksiz Tür Dönüşümleri

❌ **Yanlış Kullanım:** Tür kontrolü yaptıktan sonra manuel tür dönüşümü.

```csharp
if (obj is string)
{
    var str = (string)obj;
    Console.WriteLine(str.ToUpper());
}
```

✅ **İdeal Kullanım:** `is` operatöründe doğrudan tür dönüşümünü kullanın.

```csharp
if (obj is string str)
{
    Console.WriteLine(str.ToUpper());
}
```

---

## 2. `switch` İfadelerinde Sabit Değerler Kullanmamak

❌ **Yanlış Kullanım:** `switch` ifadelerinde sabit değerler yerine karmaşık ifadeler kullanmak.

```csharp
switch (input.Length)
{
    case int n when n > 5:
        Console.WriteLine("Uzun bir metin.");
        break;
    default:
        Console.WriteLine("Kısa bir metin.");
        break;
}
```

✅ **İdeal Kullanım:** Sabit değerler kullanarak kodun okunabilirliğini artırın.

```csharp
switch (input.Length)
{
    case > 5:
        Console.WriteLine("Uzun bir metin.");
        break;
    default:
        Console.WriteLine("Kısa bir metin.");
        break;
}
```

---

## 3. `when` Koşullarını Yanlış Kullanmak

❌ **Yanlış Kullanım:** `when` koşullarında gereksiz kontrol yapmak.

```csharp
if (obj is int i && i > 10)
{
    Console.WriteLine("10'dan büyük bir sayı.");
}
```

✅ **İdeal Kullanım:** `when` ifadesini `is` ile entegre ederek kodu sadeleştirin.

```csharp
if (obj is int i when i > 10)
{
    Console.WriteLine("10'dan büyük bir sayı.");
}
```

---

## 4. `switch` İfadelerinde Tür Kontrollerini Karmaşık Hale Getirmek

❌ **Yanlış Kullanım:** Farklı türler için ayrı `if` blokları kullanmak.

```csharp
if (obj is string str)
{
    Console.WriteLine($"Metin: {str}");
}
else if (obj is int num)
{
    Console.WriteLine($"Sayı: {num}");
}
```

✅ **İdeal Kullanım:** `switch` ifadesini kullanarak tür kontrollerini düzenleyin.

```csharp
switch (obj)
{
    case string str:
        Console.WriteLine($"Metin: {str}");
        break;
    case int num:
        Console.WriteLine($"Sayı: {num}");
        break;
    default:
        Console.WriteLine("Bilinmeyen tür.");
        break;
}
```

---

## 5. Destructuring ile Pattern Matching'i İhmal Etmek

❌ **Yanlış Kullanım:** Karmaşık veri yapılarını manuel olarak çözümlemek.

```csharp
if (point is Point)
{
    var p = (Point)point;
    Console.WriteLine($"X: {p.X}, Y: {p.Y}");
}
```

✅ **İdeal Kullanım:** Destructuring ile kodu sadeleştirin.

```csharp
if (point is Point(var x, var y))
{
    Console.WriteLine($"X: {x}, Y: {y}");
}
```

---

## 6. Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri yapılarında Pattern Matching'i optimize etmeden kullanmak.

```csharp
foreach (var item in collection)
{
    if (item is string str && str.Contains("test"))
    {
        Console.WriteLine(str);
    }
}
```

✅ **İdeal Kullanım:** Pattern Matching'i erken çıkış mekanizmaları ile birleştirin.

```csharp
foreach (var str in collection.OfType<string>())
{
    if (str.Contains("test"))
    {
        Console.WriteLine(str);
    }
}
```