# Expression Trees ile Dinamik LINQ Sorguları

Expression Trees, C# dilinde dinamik ve karmaşık sorgular oluşturmak için güçlü bir araçtır. Özellikle LINQ ile çalışırken, runtime sırasında sorgular üretmek ve bu sorguları optimize etmek için kullanılır.

---

## 1. Expression Trees Nedir?

Expression Trees, kodunuzun bir ifade biçiminde temsil edilmesini sağlar. Bu, bir sorgunun veya lambda ifadesinin analiz edilmesine ve çalışma zamanında değiştirilmesine olanak tanır.  

Özellikle aşağıdaki durumlarda kullanılır:  
- **Dinamik Filtreleme:** Kullanıcı girdilerine göre koşullar oluşturma.  
- **Veritabanı Sorguları:** LINQ to SQL veya Entity Framework ile dinamik sorgular üretme.  
- **Derin Analiz:** Lambda ifadelerini analiz ederek optimize etme.

---

## 2. Yanlış ve İdeal Kullanım

### **Yanlış Kullanım:** Statik Koşullar

❌ **Yanlış Kullanım:**

```csharp
var filteredData = data.Where(x => x.Age > 32 && x.Name == "Murat").ToList();
```

Bu sorgu, yalnızca belirli bir koşul için sabitlenmiştir ve yeniden kullanılabilirliği düşüktür.

---

### **İdeal Kullanım:** Dinamik Koşullar

✅ **İdeal Kullanım:**

```csharp
using System.Linq.Expressions;

Expression<Func<Person, bool>> filter = x => x.Age > 32 && x.Name == "Murat";
var filteredData = data.Where(filter.Compile()).ToList();
```

Bu yöntem, koşulları dinamik olarak oluşturmanıza olanak tanır.

---

## 3. Dinamik Koşullar Oluşturma

Expression Trees ile koşullar runtime sırasında oluşturulabilir.

✅ **Örnek:** Kullanıcı Girdisine Göre Dinamik Sorgu

```csharp
var parameter = Expression.Parameter(typeof(Person), "x");
var property = Expression.Property(parameter, "Age");
var constant = Expression.Constant(32);
var comparison = Expression.GreaterThan(property, constant);

var lambda = Expression.Lambda<Func<Person, bool>>(comparison, parameter);

var filteredData = data.Where(lambda.Compile()).ToList();
```

Bu kod, `x => x.Age > 32` sorgusunu runtime sırasında oluşturur.

---

## 4. Birden Fazla Koşul ile Dinamik Sorgu

Expression Trees ile birden fazla koşul dinamik olarak birleştirilebilir.

✅ **Örnek:** Dinamik Koşulların Birleştirilmesi

```csharp
var parameter = Expression.Parameter(typeof(Person), "x");

var ageProperty = Expression.Property(parameter, "Age");
var ageCondition = Expression.GreaterThan(ageProperty, Expression.Constant(32));

var nameProperty = Expression.Property(parameter, "Name");
var nameCondition = Expression.Equal(nameProperty, Expression.Constant("Murat"));

var combinedCondition = Expression.AndAlso(ageCondition, nameCondition);

var lambda = Expression.Lambda<Func<Person, bool>>(combinedCondition, parameter);

var filteredData = data.Where(lambda.Compile()).ToList();
```

Bu kod, `x => x.Age > 32 && x.Name == "Murat"` sorgusunu oluşturur.

---

## 5. Dinamik Sıralama (Dynamic Ordering)

Expression Trees kullanarak sıralama işlemleri dinamik hale getirilebilir.

✅ **Örnek:** Dinamik Sıralama

```csharp
var parameter = Expression.Parameter(typeof(Person), "x");
var property = Expression.Property(parameter, "Name");

var lambda = Expression.Lambda<Func<Person, string>>(property, parameter);

var sortedData = data.OrderBy(lambda.Compile()).ToList();
```

Bu kod, kişileri adlarına göre sıralar.

---

## 6. Performans ve Optimizasyon

Expression Trees, doğru kullanıldığında performansı artırabilir. Ancak, yanlış kullanımı durumunda gereksiz karmaşıklık ve yavaşlama yaratabilir. **Koşullar:**  
- Çok sık kullanılan sorgular önceden derlenmeli.  
- Dinamik koşullar karmaşık hale gelirse sorgu analiz edilmeli.

✅ **Performans İpucu:**

```csharp
var compiledLambda = lambda.Compile();
// Compiled sorguyu tekrar kullanabilirsiniz
var filteredData = data.Where(compiledLambda).ToList();
```