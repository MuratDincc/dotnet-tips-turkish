# Benchmarking

Benchmarking, kodun performansını ölçülebilir şekilde analiz eder; yanlış ölçüm yöntemleri yanıltıcı sonuçlara ve hatalı optimizasyona yol açar.

---

## 1. Stopwatch ile Güvenilmez Ölçüm

❌ **Yanlış Kullanım:** Manuel Stopwatch ile tek seferlik ölçüm yapmak.

```csharp
var sw = Stopwatch.StartNew();
var result = SortArray(data);
sw.Stop();
Console.WriteLine($"Süre: {sw.ElapsedMilliseconds}ms");
// JIT warm-up dahil, GC etkisi belirsiz, tek ölçüm istatistiksel anlamsız
```

✅ **İdeal Kullanım:** BenchmarkDotNet ile güvenilir ölçüm yapın.

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class SortBenchmarks
{
    private int[] _data;

    [GlobalSetup]
    public void Setup()
    {
        _data = Enumerable.Range(0, 10000).OrderByDescending(x => x).ToArray();
    }

    [Benchmark(Baseline = true)]
    public int[] ArraySort()
    {
        var copy = _data.ToArray();
        Array.Sort(copy);
        return copy;
    }

    [Benchmark]
    public int[] LinqOrderBy()
    {
        return _data.OrderBy(x => x).ToArray();
    }

    [Benchmark]
    public int[] SpanSort()
    {
        var copy = _data.ToArray();
        copy.AsSpan().Sort();
        return copy;
    }
}
```

---

## 2. Memory Allocation'ı Ölçmemek

❌ **Yanlış Kullanım:** Sadece süre ölçüp bellek tüketimini görmezden gelmek.

```csharp
[Benchmark]
public string ConcatStrings()
{
    var result = "";
    for (int i = 0; i < 1000; i++)
        result += i.ToString(); // Her iterasyonda allocation, GC baskısı
    return result;
}
```

✅ **İdeal Kullanım:** MemoryDiagnoser ile allocation'ı da ölçün.

```csharp
[MemoryDiagnoser]
public class StringBenchmarks
{
    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        var result = "";
        for (int i = 0; i < 1000; i++)
            result += i.ToString();
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < 1000; i++)
            sb.Append(i);
        return sb.ToString();
    }

    [Benchmark]
    public string StringCreate()
    {
        return string.Join("", Enumerable.Range(0, 1000));
    }
}
// MemoryDiagnoser allocation miktarını gösterir:
// Concatenation: 2,000,000 B allocated
// StringBuilder: 12,000 B allocated
```

---

## 3. Parametre Varyasyonlarını Test Etmemek

❌ **Yanlış Kullanım:** Tek veri boyutuyla benchmark yapmak.

```csharp
[Benchmark]
public void SearchList()
{
    var list = Enumerable.Range(0, 1000).ToList();
    list.Contains(999);
}
```

✅ **İdeal Kullanım:** Farklı veri boyutlarını parametrize edin.

```csharp
[MemoryDiagnoser]
public class SearchBenchmarks
{
    [Params(100, 1_000, 10_000, 100_000)]
    public int Size { get; set; }

    private List<int> _list;
    private HashSet<int> _set;

    [GlobalSetup]
    public void Setup()
    {
        _list = Enumerable.Range(0, Size).ToList();
        _set = new HashSet<int>(_list);
    }

    [Benchmark(Baseline = true)]
    public bool ListContains() => _list.Contains(Size - 1);

    [Benchmark]
    public bool HashSetContains() => _set.Contains(Size - 1);
}
```

---

## 4. Micro-Benchmark Sonuçlarını Yanlış Yorumlamak

❌ **Yanlış Kullanım:** Nanosaniye farkı olan optimizasyonu production'a uygulamak.

```csharp
// Benchmark: MethodA 5ns, MethodB 8ns
// "MethodA %37 daha hızlı!" - ama saniyede milyonlarca çağrı yoksa fark edilmez
```

✅ **İdeal Kullanım:** Hot path'i belirleyip anlamlı optimizasyona odaklanın.

```csharp
// Önce profiler ile hot path'i belirleyin
// Sonra o noktayı benchmark'layın

[MemoryDiagnoser]
public class HotPathBenchmark
{
    [Benchmark]
    public async Task<List<OrderDto>> GetOrders_WithEagerLoading()
    {
        // Gerçek senaryo: N+1 query sorunu
        return await _context.Orders
            .Include(o => o.Items)
            .Select(o => new OrderDto(o.Id, o.TotalPrice))
            .ToListAsync();
    }

    [Benchmark]
    public async Task<List<OrderDto>> GetOrders_WithProjection()
    {
        // Optimize: Sadece gerekli kolonlar
        return await _context.Orders
            .Select(o => new OrderDto(o.Id, o.TotalPrice))
            .ToListAsync();
    }
}
```

---

## 5. Benchmark Ortamını Kontrol Etmemek

❌ **Yanlış Kullanım:** Debug modda benchmark çalıştırmak.

```bash
dotnet run # Debug build, JIT optimizasyonları kapalı
```

✅ **İdeal Kullanım:** Release modda ve kontrollü ortamda çalıştırın.

```csharp
// Program.cs
BenchmarkRunner.Run<MyBenchmarks>();

// Terminal
// dotnet run -c Release

// Veya belirli benchmark'ları filtreleyin
BenchmarkRunner.Run<MyBenchmarks>(
    DefaultConfig.Instance
        .WithOptions(ConfigOptions.DisableOptimizationsValidator)
        .AddDiagnoser(MemoryDiagnoser.Default));
```