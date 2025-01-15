# Garbage Collector: Bellek Sızıntısı Tespiti

Bellek sızıntıları, bir uygulamanın gereğinden fazla bellek tüketmesine ve zamanla performans düşüşüne neden olabilir. .NET'in Garbage Collector (GC) mekanizması genellikle bellek yönetimini otomatik olarak yapar, ancak yanlış referans yönetimi veya karmaşık nesne ilişkileri bellek sızıntılarına yol açabilir.

---

## 1. Bellek Sızıntısı Nedir?

Bellek sızıntısı, artık kullanılmayan ancak Garbage Collector tarafından serbest bırakılmayan nesnelerin bellek tüketmeye devam etmesi durumudur. Bu, genellikle aşağıdaki nedenlerden kaynaklanır:

- **Yanlış Referans Yönetimi:** Gereksiz referansların tutulması.
- **Olay (Event) Abonelikleri:** `event` aboneliklerinin iptal edilmemesi.
- **Statik Nesneler:** Statik alanlarda gereksiz veri tutulması.

---

## 2. Bellek Sızıntısı Nasıl Tespit Edilir?

.NET uygulamalarında bellek sızıntılarını tespit etmek için şu araçları kullanabilirsiniz:

### Visual Studio Diagnostic Tools
- **Memory Usage:** Uygulamanın bellek kullanımını izler.
- **Heap Snapshots:** Anlık bellek durumlarını karşılaştırır.

### .NET CLI Tools
- **dotnet-dump:** Bellek dökümleri oluşturur ve analiz eder.
- **dotnet-counters:** Gerçek zamanlı bellek ölçümleri sağlar.

```bash
dotnet-dump collect --process-id <pid>
dotnet-counters monitor --counters Microsoft-Windows-DotNETRuntime:GC/Heap
```

---

## 3. Bellek Sızıntısı Nedenleri ve Çözümleri

### Yanlış Referans Yönetimi

❌ **Yanlış Kullanım:** Gereksiz referansları temizlememek.

```csharp
static List<byte[]> cache = new();

void AddToCache()
{
    var data = new byte[1024 * 1024];
    cache.Add(data); // Referans bırakılmıyor
}
```

✅ **İdeal Kullanım:** Referansları zamanında serbest bırakmak.

```csharp
static List<byte[]> cache = new();

void ClearCache()
{
    cache.Clear(); // Referanslar serbest bırakılır
}
```

---

### Event Aboneliklerini Yönetmemek

❌ **Yanlış Kullanım:** Olaylara abone olduktan sonra iptal etmemek.

```csharp
button.Click += OnButtonClick; // Abonelik iptal edilmez
```

✅ **İdeal Kullanım:** Olay aboneliklerini iptal edin.

```csharp
button.Click -= OnButtonClick; // Abonelik iptal edilir
```

---

### Statik Alanlarda Veri Tutmak

❌ **Yanlış Kullanım:** Statik alanlarda büyük nesneleri gereksiz tutmak.

```csharp
static List<int> staticData = new() { 1, 2, 3 };
```

✅ **İdeal Kullanım:** Statik alanları dikkatli yönetin.

```csharp
static WeakReference<List<int>> staticData = new(new List<int> { 1, 2, 3 });
```

---

## 4. Garbage Collector Diagnostik Modu

.NET 9 ile gelen diagnostik araçlar, GC'nin bellek yönetimini analiz etmeyi kolaylaştırır.

```xml
<configuration>
  <runtime>
    <GCHeapHardLimitPercent value="80" />
  </runtime>
</configuration>
```

---

## 5. Profiling ve İzleme

- **JetBrains dotMemory:** Derinlemesine bellek analizi.
- **Redgate ANTS Memory Profiler:** Hafıza sızıntılarını tespit etmek için kullanılır.