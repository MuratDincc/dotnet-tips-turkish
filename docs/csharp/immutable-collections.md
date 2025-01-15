# Immutable Collections

Immutable collections, veri yapılarında değişiklik yapılmasını engelleyerek veri tutarlılığını sağlar. Performans avantajları ve eş zamanlı işlemlerde güvenilirlik sunar. Ancak, yanlış kullanımları gereksiz bellek tüketimine ve karmaşıklığa yol açabilir.

---

## 1. Immutable Collections Yerine Mutable Collections Kullanmak

❌ **Yanlış Kullanım:** Veri tutarlılığı gerektiren durumlarda mutable koleksiyonları kullanmak.

```csharp
var list = new List<int> { 1, 2, 3 };
list.Add(4); // Liste değiştirilebilir
Console.WriteLine(string.Join(", ", list));
```

✅ **İdeal Kullanım:** Değiştirilemez bir koleksiyon kullanarak veri tutarlılığını koruyun.

```csharp
var immutableList = ImmutableList.Create(1, 2, 3);
var newList = immutableList.Add(4); // Yeni bir koleksiyon oluşturulur
Console.WriteLine(string.Join(", ", newList));
```

---

## 2. Performansı Göz Ardı Etmek

❌ **Yanlış Kullanım:** Büyük veri kümelerinde gereksiz immutable dönüşümler yapmak.

```csharp
var list = new List<int>();
for (int i = 0; i < 1000; i++)
{
    list = list.Append(i).ToList(); // Her dönüşümde yeni bir liste oluşturulur
}
```

✅ **İdeal Kullanım:** Immutable koleksiyonlarla performans dostu işlemler gerçekleştirin.

```csharp
var builder = ImmutableList.CreateBuilder<int>();
for (int i = 0; i < 1000; i++)
{
    builder.Add(i);
}
var immutableList = builder.ToImmutable();
Console.WriteLine(string.Join(", ", immutableList));
```

---

## 3. Immutable Collections'ı Yanlış Anlamak

❌ **Yanlış Kullanım:** Immutable koleksiyonun mevcut koleksiyonu değiştirdiğini düşünmek.

```csharp
var immutableList = ImmutableList.Create(1, 2, 3);
immutableList.Add(4); // Yeni koleksiyon oluşturur, ama atama yapılmaz
Console.WriteLine(string.Join(", ", immutableList)); // Eski liste
```

✅ **İdeal Kullanım:** Değişiklikleri yeni bir koleksiyona atayın.

```csharp
var immutableList = ImmutableList.Create(1, 2, 3);
var updatedList = immutableList.Add(4);
Console.WriteLine(string.Join(", ", updatedList)); // Güncellenmiş liste
```

---

## 4. Gereksiz Veri Kopyalama

❌ **Yanlış Kullanım:** Her işlemde yeni bir immutable koleksiyon oluşturmak.

```csharp
var immutableList = ImmutableList.Create<int>();
for (int i = 0; i < 100; i++)
{
    immutableList = immutableList.Add(i); // Her seferinde yeni bir liste oluşturulur
}
```

✅ **İdeal Kullanım:** Builder nesnesi ile gereksiz kopyalamalardan kaçının.

```csharp
var builder = ImmutableList.CreateBuilder<int>();
for (int i = 0; i < 100; i++)
{
    builder.Add(i);
}
var immutableList = builder.ToImmutable();
Console.WriteLine(string.Join(", ", immutableList));
```

---

## 5. Yanlış Koleksiyon Türünü Kullanmak

❌ **Yanlış Kullanım:** İhtiyaca uygun olmayan immutable koleksiyon türlerini kullanmak.

```csharp
var immutableStack = ImmutableStack<int>.Empty;
immutableStack = immutableStack.Push(1);
immutableStack = immutableStack.Push(2);
Console.WriteLine(immutableStack.Peek()); // 2
```

✅ **İdeal Kullanım:** İşlem türüne uygun immutable koleksiyon seçin.

```csharp
var immutableQueue = ImmutableQueue<int>.Empty;
immutableQueue = immutableQueue.Enqueue(1);
immutableQueue = immutableQueue.Enqueue(2);
Console.WriteLine(immutableQueue.Peek()); // 1
```