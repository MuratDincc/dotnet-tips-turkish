# Expression-Bodied Members

Expression-bodied members, C# dilinde kısa ve öz kod yazmayı sağlar. Ancak, yanlış kullanım durumları kodun okunabilirliğini ve sürdürülebilirliğini etkileyebilir.

---

## 1. Gereksiz Yerine Getirme Yöntemleri

❌ **Yanlış Kullanım:** Basit dönüşleri tam metot gövdesiyle yazmak.

```csharp
public string GetName()
{
    return "Murat";
}
```

✅ **İdeal Kullanım:** Expression-bodied members ile metotları sadeleştirin.

```csharp
public string GetName() => "Murat";
```

---

## 2. Karmaşık İfadeleri Expression-Bodied Members ile Yazmak

❌ **Yanlış Kullanım:** Çok satırlı ifadeleri tek bir expression-bodied member ile yazmak.

```csharp
public string GetFullName(string firstName, string lastName) => 
    $"{firstName} {lastName}".ToUpper() + $" Length: {firstName.Length + lastName.Length}";
```

✅ **İdeal Kullanım:** Karmaşık ifadeleri birden fazla satırda açıkça yazın.

```csharp
public string GetFullName(string firstName, string lastName)
{
    var fullName = $"{firstName} {lastName}";
    return $"{fullName.ToUpper()} Length: {fullName.Length}";
}
```

---

## 3. Kapsamlı Property Gövdeleri Kullanmak

❌ **Yanlış Kullanım:** Property'ler için tam gövde kullanmak.

```csharp
private string _name;

public string Name
{
    get { return _name; }
    set { _name = value; }
}
```

✅ **İdeal Kullanım:** Expression-bodied members ile property'leri kısaltın.

```csharp
public string Name { get; set; }
```

---

## 4. Exception Fırlatma İşlemlerinde Expression-Bodied Kullanımı

❌ **Yanlış Kullanım:** Exception fırlatma işlemlerini expression-bodied members ile karmaşık hale getirmek.

```csharp
public string Name => throw new ArgumentNullException(nameof(Name), "Name is required.");
```

✅ **İdeal Kullanım:** Exception işlemlerini açık bir şekilde yazın.

```csharp
public string Name
{
    get => throw new ArgumentNullException(nameof(Name), "Name is required.");
}
```

---

## 5. Constructor'larda Expression-Bodied Kullanımını Yanlış Yapmak

❌ **Yanlış Kullanım:** Constructor'ları gereksiz yere expression-bodied olarak yazmak.

```csharp
public Person(string name) => Name = name ?? throw new ArgumentNullException(nameof(name));
```

✅ **İdeal Kullanım:** Constructor'larda expression-bodied kullanımını sade tutun.

```csharp
public Person(string name)
{
    Name = name ?? throw new ArgumentNullException(nameof(name));
}
```

---

## 6. Basit İşlemleri Açık Gövdelerle Yazmak

❌ **Yanlış Kullanım:** Basit property'ler için tam metot gövdesi kullanmak.

```csharp
public string Description
{
    get { return "A short description."; }
}
```

✅ **İdeal Kullanım:** Expression-bodied members ile basit işlemleri optimize edin.

```csharp
public string Description => "A short description.";
```