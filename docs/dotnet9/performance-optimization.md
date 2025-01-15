# Performans Optimizasyonu

.NET 9, performans odaklı iyileştirmeler ve araçlarla birlikte geliyor.

---

## 1. AOT (Ahead-of-Time) Compilation

AOT derleme, uygulamanızın çalışma zamanında değil, derleme sırasında derlenmesini sağlar. Bu, özellikle başlangıç süresini ve bellek kullanımını optimize eder.

### Kullanım
```bash
dotnet publish -c Release -r linux-x64 --self-contained
```

AOT ile çalışan uygulamalarınızda başlangıç performansında büyük bir artış gözlemleyebilirsiniz.

---

## 2. Dynamic PGO (Profile Guided Optimization)

Dynamic PGO, uygulamanızın çalışma zamanındaki kullanım profillerine göre kodu optimize eder.


```csharp
Console.WriteLine("Dynamic PGO ile optimize edildi!");
```

Bu özellik, kodunuzun sık kullanılan bölümlerini optimize ederek performansı artırır.

---

## 3. Span<T> ve Memory<T> ile Hafif Veri İşleme

`Span<T>` ve `Memory<T>`, veri işleme sırasında gereksiz kopyalamaları azaltır ve bellek tahsisini optimize eder.


```csharp
Span<int> numbers = stackalloc int[] { 1, 2, 3, 4, 5 };
var slice = numbers.Slice(1, 3);

foreach (var number in slice)
{
    Console.WriteLine(number); // Çıktı: 2, 3, 4
}
```

---

## 4. Garbage Collector İyileştirmeleri

.NET 9, Garbage Collector (GC) için önemli iyileştirmeler sunar. Özellikle `Gen0` ve `Gen1` koleksiyonlarında daha hızlı temizlik yapılır.

### İpucu: GC Ayarlarını Optimize Etmek
```csharp
GCSettings.LatencyMode = GCLatencyMode.LowLatency;
```

Bu ayar, zaman duyarlı işlemler için GC'nin davranışını optimize eder.

---

## 5. Asenkron Programlama

Asenkron programlama, uzun süren işlemleri engellemeden daha verimli bir uygulama sağlar.

### IAsyncEnumerable Kullanımı
```csharp
public async IAsyncEnumerable<int> GetNumbersAsync()
{
    for (int i = 0; i < 5; i++)
    {
        await Task.Delay(500);
        yield return i;
    }
}
```

---

## 6. Benchmark.NET ile Performans Analizi

Benchmark.NET, performans analizi yapmanıza olanak tanır. Kodunuzun hangi bölümlerinin yavaş olduğunu anlamak için mükemmel bir araçtır.


```csharp
[MemoryDiagnoser]
public class Benchmarks
{
    [Benchmark]
    public void StringConcat()
    {
        string result = string.Empty;
        for (int i = 0; i < 1000; i++)
        {
            result += i;
        }
    }
}
```

```bash
dotnet run -c Release
```

---

## 7. SIMD ve Vectorization

SIMD (Single Instruction Multiple Data) desteği, paralel veri işleme için kullanılabilir.


```csharp
using System.Numerics;

Vector<int> vector1 = new Vector<int>(new int[] { 1, 2, 3, 4 });
Vector<int> vector2 = new Vector<int>(new int[] { 5, 6, 7, 8 });

Vector<int> result = vector1 + vector2;

Console.WriteLine(result); // Çıktı: <6, 8, 10, 12>
```

---

## 8. Dosya IO Performansı

.NET 9, dosya IO işlemlerinde performans iyileştirmeleri sunar.


```csharp
await using var stream = new FileStream("example.txt", FileMode.Open);
byte[] buffer = new byte[1024];
await stream.ReadAsync(buffer, 0, buffer.Length);
```

---

## İlgili Kaynaklar

- [AOT Compilation in .NET](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run)
- [Span<T> Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.span-1)
- [Benchmark.NET Documentation](https://benchmarkdotnet.org/)