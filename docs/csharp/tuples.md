# Tuples

C# dilinde tuples, birden fazla değeri bir arada taşımak için kullanışlı bir veri yapısıdır. Ancak, yanlış kullanım durumları kodun okunabilirliğini ve sürdürülebilirliğini azaltabilir.

---

## 1. Anlamsız Tuple İsimleri Kullanmak

❌ **Yanlış Kullanım:** Tuple bileşenlerini varsayılan isimlerle bırakmak.

```csharp
var result = GetPerson();
Console.WriteLine(result.Item1); // Anlamsız
Console.WriteLine(result.Item2);
```

✅ **İdeal Kullanım:** Tuple bileşenlerine anlamlı isimler verin.

```csharp
var (name, age) = GetPerson();
Console.WriteLine(name);
Console.WriteLine(age);
```

**Tanım:**
```csharp
(string Name, int Age) GetPerson() => ("Murat", 33);
```

---

## 2. Tuples Yerine Sınıfları Kullanmayı İhmal Etmek

❌ **Yanlış Kullanım:** Karmaşık veri yapıları için tuple kullanmak.

```csharp
(string, int, string) GetDetailedPerson() => ("Murat", 33, "Istanbul");
```

✅ **İdeal Kullanım:** Daha karmaşık veri yapıları için sınıf veya kayıt yapısı kullanın.

```csharp
public record Person(string Name, int Age, string City);

Person GetDetailedPerson() => new("Murat", 33, "Istanbul");
```

---

## 3. Uzun Tuple Yapıları Kullanmak

❌ **Yanlış Kullanım:** Fazla sayıda bileşen içeren tuple tanımları.

```csharp
(string, int, string, string, bool) GetComplexData() => ("Murat", 33, "Istanbul", "Yazılım Mimarı", true);
```

✅ **İdeal Kullanım:** Daha kısa ve anlamlı tuple yapıları kullanın.

```csharp
(string Name, int Age) GetBasicData() => ("Murat", 33);
```

---

## 4. Tuple'ları Döngülerde Yanlış Kullanmak

❌ **Yanlış Kullanım:** Tuple bileşenlerine doğrudan indeks ile erişmek.

```csharp
var data = new List<(string, int)>
{
    ("Murat", 33),
    ("Derin", 2)
};

foreach (var item in data)
{
    Console.WriteLine($"Name: {item.Item1}, Age: {item.Item2}");
}
```

✅ **İdeal Kullanım:** Tuple bileşenlerini anlamlı isimlerle kullanın.

```csharp
var data = new List<(string Name, int Age)>
{
    ("Murat", 33),
    ("Derin", 2)
};

foreach (var (name, age) in data)
{
    Console.WriteLine($"Name: {name}, Age: {age}");
}
```

---

## 5. Tuple'ları Geri Dönüş Değeri Olarak Yanlış Kullanmak

❌ **Yanlış Kullanım:** Açıkça tanımlanmamış tuple'ları metot dönüş değeri olarak kullanmak.

```csharp
public (string, int) GetPerson() => ("Murat", 33);
```

✅ **İdeal Kullanım:** Tuple dönüş değerlerini açıkça tanımlayın.

```csharp
public (string Name, int Age) GetPerson() => ("Murat", 33);
```

---

## 6. Tuple Değerlerini Yanlış Değerlendirmek

❌ **Yanlış Kullanım:** Tuple'ları karşılaştırırken tüm bileşenleri kontrol etmemek.

```csharp
var tuple1 = ("Murat", 33);
var tuple2 = ("Murat", 2);

if (tuple1 == tuple2) // Derleme hatası
{
    Console.WriteLine("Eşit!");
}
```

✅ **İdeal Kullanım:** Tuple karşılaştırmalarında tüm bileşenleri dikkate alın.

```csharp
var tuple1 = ("Murat", 33);
var tuple2 = ("Murat", 33);

if (tuple1 == tuple2)
{
    Console.WriteLine("Eşit!");
}
```