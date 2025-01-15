# Garbage Collector: Çalışma Mekanizması

Garbage Collector (GC), .NET uygulamalarında bellek yönetimini otomatikleştiren önemli bir bileşendir. GC, kullanılmayan nesneleri tespit ederek belleği temizler ve bellek sızıntılarını önler. Bu, geliştiricilerin manuel bellek yönetimi yapma zorunluluğunu ortadan kaldırır. Ancak, GC'nin çalışma prensiplerini anlamadan yazılan kod performans sorunlarına yol açabilir.

---

## 1. GC'nin Çalışma Prensibi

Garbage Collector, üç temel aşamada çalışır:

1. **Marking (İşaretleme):** Kullanılan ve kullanılmayan nesneler belirlenir.
2. **Relocating (Yer Değiştirme):** Kullanılan nesneler bir araya toplanır.
3. **Compacting (Sıkıştırma):** Bellek alanı yeniden düzenlenir.

GC, bu işlemleri nesneleri nesil (generation) temelli bir sistemle yönetir.

---

## 2. Nesiller (Generations)

Garbage Collector, bellek yönetimini optimize etmek için nesneleri üç farklı nesilde yönetir:

- **Generation 0:** Kısa ömürlü nesneler (örneğin, yerel değişkenler) için kullanılır.
- **Generation 1:** Generation 0'dan terfi eden nesneler.
- **Generation 2:** Uzun ömürlü nesneler (örneğin, statik nesneler).

**Neden Nesiller?**  
GC, kısa ömürlü nesnelerin daha sık temizlendiği, uzun ömürlü nesnelerin daha az sık temizlendiği bir strateji kullanarak performansı artırır.

---

## 3. GC'nin İşlem Türleri

GC, iki ana modda çalışabilir:

1. **Workstation Mode:** Tek kullanıcılı uygulamalar için optimize edilmiştir.
2. **Server Mode:** Çok iş parçacıklı ve yüksek performans gereksinimleri olan uygulamalar için optimize edilmiştir.

---

## 4. GC'nin Performansa Etkisi

❌ **Yanlış Kullanım:** Gereksiz büyük nesne tahsisleri.

```csharp
var largeArray = new byte[1024 * 1024 * 100]; // Büyük nesneler LOH'yi etkiler
```

✅ **İdeal Kullanım:** Büyük nesnelerden kaçının veya gerektiğinde yeniden kullanın.

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024 * 1024);
try
{
    // Kullanım
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

---

## 5. Finalizer ve IDisposable Kullanımı

❌ **Yanlış Kullanım:** Finalizer ile bellek yönetimi yapmaya çalışmak.

```csharp
~MyClass()
{
    // Temizleme işlemleri
}
```

✅ **İdeal Kullanım:** IDisposable arayüzünü uygulayın ve `using` yapısını kullanın.

```csharp
public class MyClass : IDisposable
{
    public void Dispose()
    {
        // Kaynak temizleme
    }
}

using var myObject = new MyClass();
```

---

## 6. `GC.Collect` Kullanımı

❌ **Yanlış Kullanım:** Manuel olarak `GC.Collect` çağırmak.

```csharp
GC.Collect(); // Performans sorunlarına neden olabilir
```

✅ **İdeal Kullanım:** GC'yi otomatik çalıştırmasına izin verin.

```csharp
// GC'nin çalışma zamanına güvenin.
```

---

## 7. Büyük Nesne Yığını (LOH) Yönetimi

Büyük nesneler (85 KB'den büyük) Large Object Heap (LOH) üzerinde depolanır ve GC tarafından sıkıştırılmaz.

❌ **Yanlış Kullanım:** Gereksiz büyük nesneler oluşturmak.

```csharp
var data = new byte[1024 * 1024 * 10]; // LOH üzerinde büyük tahsis
```

✅ **İdeal Kullanım:** Daha küçük parçalar halinde işlem yapın.

```csharp
var chunks = new List<byte[]>();
for (int i = 0; i < 10; i++)
{
    chunks.Add(new byte[1024 * 1024]); // Daha küçük parçalara bölünmüş nesneler
}
```