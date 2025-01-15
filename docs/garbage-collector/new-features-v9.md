# Garbage Collector: .NET 9'daki Yeni Özellikler

.NET 9, Garbage Collector (GC) üzerinde çeşitli iyileştirmeler ve yeni özellikler sunarak bellek yönetimi performansını daha da geliştirmiştir. Bu özellikler, büyük ölçekli uygulamalarda daha iyi verimlilik ve daha düşük gecikme süreleri sağlamayı amaçlar.

---

## 1. Dynamic PGO ile Bellek Yönetimi

**Dynamic Profile-Guided Optimization (Dynamic PGO)**, GC'nin çalışma zamanında uygulamanızın davranışına göre kendisini optimize etmesine olanak tanır.

✅ **Yararları:**
- Gerçek zamanlı optimizasyon.
- Daha iyi bellek tahsisi.
- Performans artışı.

```csharp
// Dynamic PGO'nun otomatik etkin olduğu bir uygulama örneği
void PerformTask()
{
    var data = new List<int>();
    for (int i = 0; i < 1000; i++)
    {
        data.Add(i);
    }
    // GC daha verimli tahsis ve toplama yapar.
}
```

---

## 2. LOH (Large Object Heap) Sıkıştırma

.NET 9, Large Object Heap (LOH) için sıkıştırma özelliği ekleyerek büyük nesnelerin daha verimli bir şekilde yönetilmesini sağlar.

❌ **Önceki Davranış:**
LOH üzerinde büyük nesneler sıkıştırılmadan bırakılıyordu, bu da bellek parçalanmasına yol açabiliyordu.

✅ **Yeni Özellik:**
LOH sıkıştırma, bellek parçalanmasını azaltır.

```xml
<configuration>
  <runtime>
    <GCLOHCompact enabled="true" />
  </runtime>
</configuration>
```

---

## 3. GC'nin Daha İyi Thread Yönetimi

.NET 9, Garbage Collector'ın thread yönetimi algoritmalarında geliştirmeler yapmıştır. Bu iyileştirmeler, özellikle çok iş parçacıklı sunucu uygulamalarında daha düşük gecikme sağlar.

```csharp
<configuration>
  <runtime>
    <gcServer enabled="true" />
  </runtime>
</configuration>
```

---

## 4. Bölgesel Bellek Yönetimi (Regional GC)

.NET 9, GC'nin bellek yönetimini daha bölgesel hale getirerek bellek tahsisini hızlandırır ve gecikme sürelerini azaltır.

✅ **Avantajlar:**
- Daha küçük bellek blokları üzerinde işlem yapma.
- Bellek tahsis süresinin azalması.

---

## 5. Daha İyi Diagnostik ve İzleme Araçları

.NET 9, Garbage Collector performansını izlemek için geliştirilmiş diagnostik araçlar sunar. Bu araçlar sayesinde GC'nin nasıl çalıştığını daha detaylı bir şekilde anlayabilirsiniz.

```bash
dotnet-counters monitor --process-id <pid> --counters Microsoft-Windows-DotNETRuntime:GC/Heap
```

---

## 6. GC Performans Modu Seçenekleri

.NET 9, uygulama ihtiyaçlarına göre farklı GC modları sunar:

- **Interactive Mode:** Kullanıcı odaklı uygulamalar için düşük gecikme sağlar.
- **Batch Mode:** Sunucu odaklı uygulamalarda daha yüksek throughput için optimize edilmiştir.

```xml
<configuration>
  <runtime>
    <GCLatencyMode value="Interactive" />
  </runtime>
</configuration>
```

---

## 7. Gelişmiş İş Parçacığı Optimizasyonu

.NET 9, GC'nin iş parçacıklarını daha etkin kullanabilmesi için gelişmiş algoritmalar sunar. Bu, özellikle çok çekirdekli işlemcilerde performans artışı sağlar.

```csharp
ThreadPool.SetMinThreads(10, 10);
```