# String Interpolation

String interpolation, metin ve değişkenleri birleştirmek için etkili bir yöntem sunar. Bu özellik, kodunuzu daha okunabilir ve kısa hale getirebilir. Ancak, yanlış kullanımlar performans sorunlarına ve okunabilirlik zorluklarına yol açabilir.

---

## 1. Karmaşık İfadeleri String Interpolation'da Kullanmak

❌ **Yanlış Kullanım:** String interpolation içinde karmaşık ifadeler kullanmak.

```csharp
var name = "Murat";
var greeting = $"Merhaba, {name.ToUpper() + "!"} It is {DateTime.Now.ToString("HH:mm:ss")}";
Console.WriteLine(greeting);
```

✅ **İdeal Kullanım:** Karmaşık ifadeleri interpolation dışında işleyin.

```csharp
var name = "Murat".ToUpper();
var time = DateTime.Now.ToString("HH:mm:ss");
var greeting = $"Merhaba, {name}! It is {time}";
Console.WriteLine(greeting);
```

---

## 2. Gereksiz String.Format Kullanımı

❌ **Yanlış Kullanım:** String interpolation yerine gereksiz `string.Format` kullanımı.

```csharp
var name = "Murat";
var age = 33;
var message = string.Format("Ad: {0}, Yas: {1}", name, age);
Console.WriteLine(message);
```

✅ **İdeal Kullanım:** String interpolation ile daha temiz bir yapı kullanın.

```csharp
var name = "Murat";
var age = 33;
var message = $"Ad: {name}, Yas: {age}";
Console.WriteLine(message);
```

---

## 3. Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük döngülerde string interpolation kullanarak performansı göz ardı etmek.

```csharp
for (int i = 0; i < 1000; i++)
{
    var message = $"Current value is: {i}";
    Console.WriteLine(message);
}
```

✅ **İdeal Kullanım:** StringBuilder gibi performans dostu çözümler kullanın.

```csharp
var builder = new StringBuilder();
for (int i = 0; i < 1000; i++)
{
    builder.AppendLine($"Current value is: {i}");
}
Console.WriteLine(builder.ToString());
```

---

## 4. Kültür Farklılıklarını Göz Ardı Etmek

❌ **Yanlış Kullanım:** String interpolation'da kültür farklılıklarını dikkate almamak.

```csharp
var price = 1234.56;
var message = $"Price: {price}";
Console.WriteLine(message); // Farklı kültürlerde yanlış formatta görüntülenebilir
```

✅ **İdeal Kullanım:** Belirli bir kültürü açıkça belirterek formatlayın.

```csharp
var price = 1234.56;
var message = $"Price: {price.ToString("C", CultureInfo.GetCultureInfo("en-US"))}";
Console.WriteLine(message);
```

---

## 5. Çok Satırlı String İçinde Yanlış Kullanım

❌ **Yanlış Kullanım:** Çok satırlı stringlerde string interpolation'ı düzensiz kullanmak.

```csharp
var name = "Murat";
var message = $"Merhaba, {name}
Hosgeldin!";
Console.WriteLine(message);
```

✅ **İdeal Kullanım:** Çok satırlı stringlerde düzenli bir yapı sağlayın.

```csharp
var name = "Murat";
var message = $"Merhaba, {name}
Hosgeldin!";
Console.WriteLine(message);
```

---

## 6. Gereksiz Parantez Kullanımı

❌ **Yanlış Kullanım:** Interpolation ifadelerinde gereksiz parantezler eklemek.

```csharp
var name = "Murat";
var message = $"Merhaba, {(name)}!";
Console.WriteLine(message);
```

✅ **İdeal Kullanım:** Gereksiz parantezlerden kaçının.

```csharp
var name = "Murat";
var message = $"Merhaba, {name}!";
Console.WriteLine(message);
```