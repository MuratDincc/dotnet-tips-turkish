# Extension Methods

Extension methods, mevcut sınıflara veya arabirimlere yeni metotlar eklemenin etkili bir yoludur. Ancak, yanlış kullanım durumları kodun anlaşılabilirliğini ve bakımını zorlaştırabilir.

---

## 1. Extension Methodları Yanlış Kapsamda Kullanmak

❌ **Yanlış Kullanım:** Extension method'ları gereksiz yere genel (`global`) hale getirmek.

```csharp
public static class GlobalExtensions
{
    public static string ToUpperCase(this string input) => input.ToUpper();
}
```

✅ **İdeal Kullanım:** Extension method'ları belirli bir bağlam veya amaca yönelik kapsama sınırlayın.

```csharp
public static class StringExtensions
{
    public static string ToUpperCase(this string input) => input.ToUpper();
}
```

---

## 2. Extension Methodları Yanlış Şekilde Adlandırmak

❌ **Yanlış Kullanım:** Extension method'lara anlamlı olmayan isimler vermek.

```csharp
public static string Func(this string input) => input.ToUpper();
```

✅ **İdeal Kullanım:** Extension method'lara anlamlı ve açıklayıcı isimler verin.

```csharp
public static string ToUpperCase(this string input) => input.ToUpper();
```

---

## 3. Extension Methodlarda Gereksiz Kontroller

❌ **Yanlış Kullanım:** Gerekli olmayan null kontrolleri yapmak.

```csharp
public static string SafeToUpperCase(this string input)
{
    if (input == null) return string.Empty;
    return input.ToUpper();
}
```

✅ **İdeal Kullanım:** Gereksiz kontrollerden kaçının ve kullanıcıyı yönlendirin.

```csharp
public static string ToUpperCase(this string input) => input?.ToUpper() ?? throw new ArgumentNullException(nameof(input));
```

---

## 4. Gereksiz Parametreler Kullanmak

❌ **Yanlış Kullanım:** Extension method'da gereksiz ek parametreler kullanmak.

```csharp
public static string AppendSuffix(this string input, string suffix)
{
    return input + suffix;
}
```

✅ **İdeal Kullanım:** Gereksiz parametrelerden kaçının.

```csharp
public static string AppendSuffix(this string input, string suffix = "Default")
{
    return input + suffix;
}
```

---

## 5. Extension Methodları Karmaşık Hale Getirmek

❌ **Yanlış Kullanım:** Çok fazla işlevselliği tek bir extension method'da birleştirmek.

```csharp
public static string TransformText(this string input, bool toUpper, bool addSuffix)
{
    var result = input;
    if (toUpper) result = result.ToUpper();
    if (addSuffix) result += "_Suffix";
    return result;
}
```

✅ **İdeal Kullanım:** İşlevleri birden fazla extension method'a bölün.

```csharp
public static string ToUpperCase(this string input) => input.ToUpper();

public static string AddSuffix(this string input, string suffix) => input + suffix;
```

---

## 6. Extension Methodların Kullanımını Belgelememek

❌ **Yanlış Kullanım:** Extension method'ların nasıl kullanılacağını açıklamamak.

```csharp
public static string AddPrefix(this string input, string prefix)
{
    return prefix + input;
}
```

✅ **İdeal Kullanım:** Extension method'ların kullanımını yorumlarla açıklayın.

```csharp
/// <summary>
/// Belirtilen metnin başına bir önek ekler.
/// </summary>
/// <param name="input">Orijinal metin.</param>
/// <param name="prefix">Eklenecek önek.</param>
/// <returns>Önek eklenmiş metin.</returns>
public static string AddPrefix(this string input, string prefix)
{
    return prefix + input;
}
```