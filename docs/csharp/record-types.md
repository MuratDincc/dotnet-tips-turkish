# Record Types

C# dilinde Record Types, immutable veri modelleri ve değer tabanlı eşitlik karşılaştırmaları oluşturmak için kullanılan modern bir yapıdır. Yanlış kullanım durumları Record Type'ların avantajlarını azaltabilir.

---

## 1. Record'ları Immutable Yapıda Tutmamak

❌ **Yanlış Kullanım:** Record'ların alanlarını değiştirilebilir (`mutable`) yapmak.

```csharp
public record Person
{
    public string Name { get; set; } // Değiştirilebilir
}
```

✅ **İdeal Kullanım:** Record'ları immutable yapıda tutarak veri bütünlüğünü sağlayın.

```csharp
public record Person(string Name);
```

---

## 2. Eşitlik Karşılaştırmalarını Yanlış Yapılandırmak

❌ **Yanlış Kullanım:** Eşitlik karşılaştırmaları için `class` kullanmak.

```csharp
public class Person
{
    public string Name { get; set; }
}

// Reference eşitliği kontrol edilir
var p1 = new Person { Name = "Murat" };
var p2 = new Person { Name = "Murat" };
Console.WriteLine(p1 == p2); // False
```

✅ **İdeal Kullanım:** Record Type kullanarak değer tabanlı eşitliği etkinleştirin.

```csharp
public record Person(string Name);

var p1 = new Person("Murat");
var p2 = new Person("Murat");
Console.WriteLine(p1 == p2); // True
```

---

## 3. `with` Anahtar Kelimesini Yanlış Kullanmak

❌ **Yanlış Kullanım:** `with` ifadesini kullanmadan veri değiştirmeye çalışmak.

```csharp
var person = new Person("Murat");
person.Name = "Derin"; // Derleme hatası
```

✅ **İdeal Kullanım:** `with` anahtar kelimesini kullanarak yeni bir Record örneği oluşturun.

```csharp
var person = new Person("Murat");
var updatedPerson = person with { Name = "Derin" };
```

---

## 4. Veri Modeli İçin Yanlış Yapı Seçimi

❌ **Yanlış Kullanım:** Record Type yerine `class` veya `struct` kullanmak.

```csharp
public class Address
{
    public string City { get; set; }
}
```

✅ **İdeal Kullanım:** Immutable ve değer tabanlı eşitlik gerektiren durumlar için Record Type kullanın.

```csharp
public record Address(string City);
```

---

## 5. Record'ların Performans Özelliklerini Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri yapıları için Record Type kullanmak.

```csharp
public record LargeRecord(string Data); // Performans sorunlarına neden olabilir
```

✅ **İdeal Kullanım:** Büyük veri yapıları için `class` kullanmayı değerlendirin.

```csharp
public class LargeRecord
{
    public string Data { get; set; }
}
```

---

## 6. Record'ları Yanlış Kapsamda Kullanmak

❌ **Yanlış Kullanım:** Record Type'ları DTO (Data Transfer Object) dışında kullanmak.

```csharp
public record Repository(string Name); // Yanlış kullanım, record yerine class kullanılmalı
```

✅ **İdeal Kullanım:** Record Type'ları DTO ve veri modelleri için kullanın.

```csharp
public record PersonDto(string Name, int Age);
```