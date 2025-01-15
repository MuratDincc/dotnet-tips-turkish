# Target-Typed New

C# 9.0 ile gelen target-typed `new` özelliği, tür çıkarımını kolaylaştırarak kodunuzu daha kısa ve okunabilir hale getirir. Ancak, yanlış kullanımlar kodun anlaşılabilirliğini ve bakımını zorlaştırabilir.

---

## 1. Hedef Türün Belirsiz Olduğu Durumlar

❌ **Yanlış Kullanım:** Hedef türün açık olmadığı durumlarda target-typed `new` kullanmak.

```csharp
var person = new(); // Hangi tür olduğu anlaşılamaz
person.Name = "Murat";
```

✅ **İdeal Kullanım:** Hedef türün net bir şekilde belirtildiği durumlarda kullanın.

```csharp
Person person = new();
person.Name = "Murat";
```

**Tanım:**
```csharp
public class Person
{
    public string Name { get; set; }
}
```

---

## 2. Karmaşık İfadelerde Target-Typed `new` Kullanmak

❌ **Yanlış Kullanım:** Target-typed `new`'i karmaşık ifadelerde kullanarak kodu daha az okunabilir hale getirmek.

```csharp
var person = new("Murat", 33); // Özellikle birden fazla constructor varsa belirsizlik yaratabilir
```

✅ **İdeal Kullanım:** Target-typed `new`'i basit ifadelerde kullanın.

```csharp
Person person = new("Murat", 33);
```

**Constructor Tanımı:**
```csharp
public Person(string name, int age)
{
    Name = name;
    Age = age;
}
```

---

## 3. Koleksiyonlarda Kullanımı Yanlış Yönetmek

❌ **Yanlış Kullanım:** Koleksiyon oluştururken hedef türü belirtmemek.

```csharp
var people = new List<Person>
{
    new() { Name = "Murat" },
    new() { Name = "Derin" }
};
```

✅ **İdeal Kullanım:** Koleksiyonun türünü açıkça belirtin.

```csharp
List<Person> people = new()
{
    new() { Name = "Murat" },
    new() { Name = "Derin" }
};
```

---

## 4. İsimlendirilmiş Argümanlarla Hatalı Kullanım

❌ **Yanlış Kullanım:** İsimlendirilmiş argümanlarla target-typed `new` kullanımı belirsizlik yaratabilir.

```csharp
var person = new(name: "Murat", age: 33);
```

✅ **İdeal Kullanım:** İsimlendirilmiş argümanlar kullanırken hedef türü netleştirin.

```csharp
Person person = new(name: "Murat", age: 33);
```

---

## 5. Target-Typed `new` ve `Nullable` Türler

❌ **Yanlış Kullanım:** Nullable türlerle target-typed `new` kullanımı yanlış anlaşılmalara yol açabilir.

```csharp
Person? person = new(); // Nullable ama hangi constructor çağrıldığı belirsiz
```

✅ **İdeal Kullanım:** Nullable türlerle kullanımda hedef türü netleştirin.

```csharp
Person? person = new Person();
```

---

## 6. Test Edilebilirliği Göz Ardı Etmek

❌ **Yanlış Kullanım:** Test edilebilirlik açısından target-typed `new`'in etkisini dikkate almamak.

```csharp
var service = new();
```

✅ **İdeal Kullanım:** Türü net bir şekilde belirtin ve test edilebilirliği artırın.

```csharp
IService service = new ServiceImplementation();
```

**Tanım:**
```csharp
public interface IService { }
public class ServiceImplementation : IService { }
```