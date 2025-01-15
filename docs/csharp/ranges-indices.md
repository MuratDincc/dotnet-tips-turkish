# Ranges and Indices

C# 8.0 ile tanıtılan Ranges (`..`) ve Indices (`^`) özellikleri, koleksiyonlar üzerinde daha okunabilir ve kısa işlemler yapmanızı sağlar. Ancak, bu özelliklerin yanlış kullanımı beklenmedik sonuçlara veya performans sorunlarına neden olabilir.

---

## 1. Anlamlı Kullanımı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Geleneksel yöntemlerle gereksiz karmaşık işlemler yapmak.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var lastThree = array.Skip(array.Length - 3).ToArray();
```

✅ **İdeal Kullanım:** Indices özelliğini kullanarak işlemleri basitleştirin.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var lastThree = array[^3..];
```

---

## 2. Negatif Indices Kullanımını Yanlış Anlamak

❌ **Yanlış Kullanım:** Negatif indekslerin yanlış yorumlanması.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var invalidIndex = array[-1]; // Derleme hatası
```

✅ **İdeal Kullanım:** Indices ile son elemanlara doğru erişim sağlayın.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var lastElement = array[^1]; // Son eleman
```

---

## 3. Ranges ile Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri setlerinde gereksiz kopyalamalar yapmak.

```csharp
var data = Enumerable.Range(1, 1000000).ToArray();
var subset = data.Skip(100).Take(50).ToArray(); // Gereksiz kopyalama
```

✅ **İdeal Kullanım:** Ranges ile performansı optimize edin.

```csharp
var data = Enumerable.Range(1, 1000000).ToArray();
var subset = data[100..150]; // Kopyalama minimal
```

---

## 4. Koleksiyonların Dışında Ranges Kullanımı

❌ **Yanlış Kullanım:** Ranges ve Indices özelliklerini uygun olmayan veri türlerinde kullanmak.

```csharp
string text = "Hello World";
var invalidRange = text[^5..]; // Sadece dizi ve liste türlerinde geçerli
```

✅ **İdeal Kullanım:** Ranges ve Indices özelliklerini doğru veri türleriyle kullanın.

```csharp
string text = "Hello World";
var substring = text[^5..]; // Geçerli ve etkili kullanım
```

---

## 5. Start ve End Indices'i Yanlış Tanımlamak

❌ **Yanlış Kullanım:** Başlangıç ve bitiş indekslerini karıştırmak.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var invalidRange = array[5..3]; // Hata: Bitiş indeksi başlangıçtan önce
```

✅ **İdeal Kullanım:** Ranges için doğru başlangıç ve bitiş indekslerini belirleyin.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var validRange = array[3..5]; // Doğru kullanım
```

---

## 6. Ranges ve Indices'i Birlikte Kullanmayı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Özellikleri birlikte kullanmamak.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var firstThree = array.Take(3).ToArray(); // Gereksiz karmaşık
```

✅ **İdeal Kullanım:** Indices ve Ranges'i birlikte kullanarak daha temiz bir yapı elde edin.

```csharp
var array = new int[] { 1, 2, 3, 4, 5 };
var firstThree = array[..3]; // İlk üç eleman
```