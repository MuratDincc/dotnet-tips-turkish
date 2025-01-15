# .NET 9 Yeni Özellikler

.NET 9, geliştiricilere performans, güvenlik ve modern geliştirme süreçlerinde birçok yenilik sunuyor. Bu rehber, .NET
9'un öne çıkan yeni özelliklerini ve geliştirmelerini detaylı bir şekilde ele alır.

---

## 1. AOT (Ahead-of-Time) Compilation

AOT (Ahead-of-Time) Compilation, uygulamanızın çalışma zamanında değil, derleme sırasında derlenmesini sağlar. Bu özellik, .NET 9 ile birlikte performansı artırmak, başlangıç süresini azaltmak ve native platform desteğini iyileştirmek için sunulmuştur. Özellikle başlangıç süresi kritik olan uygulamalar için büyük avantajlar sunar. Bu özellik, IoT cihazları, container tabanlı dağıtımlar ve native uygulamalar için idealdir.

---

### Avantajları
1. **Hızlı Başlangıç Süresi:** Uygulamanızın çalışma zamanında derleme yapmasını engelleyerek daha hızlı başlatılır.
2. **Daha Az Bellek Kullanımı:** JIT derleme için gereken bellek kullanımını ortadan kaldırır.
3. **Platform Uyumluluğu:** Native platformlara daha iyi destek sağlar.

---

### Temel Kullanım

Bir .NET uygulamasını AOT ile derlemek için şu komut kullanılır:

```bash
dotnet publish -c Release -r win-x64 --self-contained
```

Bu komut, uygulamanızı belirli bir platform için önceden derler ve çalıştırılabilir bir dosya oluşturur.

---

### Platforma Özgü Derleme

IoT cihazları veya düşük güçlü sistemler gibi belirli platformlarda çalışacak şekilde derleme yapmak:

```bash
dotnet publish -c Release -r linux-arm64 --self-contained
```

Bu, uygulamanızın native bir ARM64 cihazında çalışmasını sağlar.

---

### AOT ile Performans Karşılaştırması

Aşağıda, bir AOT derlemesi yapılmış ve JIT kullanılan bir uygulamanın basit karşılaştırması verilmiştir:

#### Kod Örneği:
```csharp
public class Program
{
    public static void Main()
    {
        Console.WriteLine("Performans Testi: AOT vs JIT");
        var start = DateTime.Now;
        for (int i = 0; i < 100000; i++)
        {
            Compute(i);
        }
        var end = DateTime.Now;
        Console.WriteLine($"Geçen Süre: {end - start}");
    }

    static int Compute(int value) => value * value;
}
```

AOT ile derlenen bir uygulama, JIT ile çalışan aynı uygulamadan genellikle %30-40 daha hızlı başlangıç süresine sahip olacaktır.

---

### Bellek Kullanımında İyileştirmeler

AOT, bellek yoğun uygulamalarda fayda sağlar, çünkü JIT derleme sırasında geçici olarak kullanılan bellek bloklarını ortadan kaldırır.

```bash
dotnet publish -c Release -r linux-x64 --self-contained
```

Derlenen dosyanızın boyutunu ve bellek kullanımını ölçmek için `htop` veya `top` gibi araçları kullanabilirsiniz.

---

### Sınırlamalar ve Dikkat Edilmesi Gerekenler
1. **Dosya Boyutu:** AOT ile derlenen uygulamalar genellikle daha büyük dosya boyutuna sahiptir.
2. **Dinamik Kod Kullanımı:** Reflection gibi çalışma zamanı özellikleri sınırlı veya uyumsuz olabilir.
3. **Platforma Özgü Derleme:** Belirli bir platform için derlenen uygulamalar diğer platformlarda çalışmaz.

---

## İlgili Kaynaklar

- [Microsoft Docs: AOT Compilation](https://learn.microsoft.com/en-us/dotnet/core/deploying/ready-to-run)
- [.NET Publish Documentation](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-publish)

---

## 2. Dynamic PGO (Profile Guided Optimization)

Dynamic PGO (Profil Güdümlü Optimizasyon), çalışma zamanı sırasında uygulamanızın kullanım profillerine göre kodu
optimize etme yeteneği sunar. .NET 9 ile gelen bu özellik, kodunuzun sık kullanılan bölümlerini tanır ve performans
açısından kritik yolları optimize eder.

---

### Avantajları

1. **Dinamik Optimizasyon:** Çalışma zamanındaki kullanım profiline göre optimize edilen kod.
2. **Performans Kazanımı:** Sık kullanılan kod yollarını hızlandırır.
3. **Geliştirilmiş Verimlilik:** Kodunuzun daha az kullanılan bölümleri optimize edilmez, bu da kaynak kullanımını
   iyileştirir.

---

### Temel Kullanım

Dynamic PGO, derleyicinin çalışma zamanında kodu daha akıllıca optimize etmesini sağlar.

```csharp
public int ComputeValue(int input)
{
    if (input > 100)
    {
        return input * 2;
    }
    return input / 2;
}

for (int i = 0; i < 100000; i++)
{
    ComputeValue(i);
}
```

Dynamic PGO, `input > 100` kontrolünün genellikle `false` döndüğünü fark edebilir ve bu yolu optimize eder.

---

### İleri Seviye Örnekler

#### Sık Kullanılan Yolların Optimizasyonu

```csharp
public static int CalculateDiscount(int amount)
{
    if (amount > 500)
    {
        return (int)(amount * 0.1); // Büyük indirim
    }
    return (int)(amount * 0.05); // Küçük indirim
}

public static void Main()
{
    var random = new Random();
    for (int i = 0; i < 1000000; i++)
    {
        CalculateDiscount(random.Next(1, 1000));
    }
}
```

Dynamic PGO, `amount > 500` kontrolünün sıkça `false` olduğunu fark eder ve bu durumu optimize eder.

---

#### Inline Optimization

Dynamic PGO, sık kullanılan fonksiyonları inline hale getirerek çağrı maliyetini düşürür.

```csharp
public static int AddNumbers(int a, int b)
{
    return a + b;
}

public static void Main()
{
    for (int i = 0; i < 1000000; i++)
    {
        var result = AddNumbers(i, i + 1);
    }
}
```

---

### Dynamic PGO ile Daha Fazlası

- **Multi-Threading Desteği:** Birden fazla thread'in aynı optimizasyonları paylaşmasını sağlar.
- **Profil Çıktıları:** Çalışma zamanında uygulamanın nasıl optimize edildiğini görmek için analiz araçları
  kullanılabilir.

---

Dynamic PGO, performans açısından kritik uygulamalar için güçlü bir özelliktir ve kodunuzu daha verimli hale getirmek
için derleyicinin yeteneklerini en üst düzeye çıkarır.

---

## 3. HTTP/3 Desteği

.NET 9, HTTP/3 protokolü için yerleşik destek sağlar. Bu, daha hızlı ve güvenilir bağlantılar kurmayı mümkün kılar.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps();
        listenOptions.Protocols = HttpProtocols.Http3;
    });
});

var app = builder.Build();
app.Run();
```

---

## 4. Source Generators Geliştirmeleri

Source Generators, derleme sırasında kod üretmeyi kolaylaştırır. .NET 9, daha iyi performans ve kullanım kolaylığı
sağlayan geliştirmeler sunar.

```csharp
[MyCustomGenerator]
public partial class ExampleClass
{
    // Otomatik oluşturulan kod buraya eklenecek
}
```

---

## 5. File IO Performans İyileştirmeleri

Dosya IO işlemlerinde daha hızlı okuma ve yazma performansı sağlar. Asenkron IO işlemleri, kaynak yönetimini daha
verimli hale getirir.

```csharp
await using var stream = new FileStream("example.txt", FileMode.Open);
byte[] buffer = new byte[1024];
await stream.ReadAsync(buffer, 0, buffer.Length);
```

---

## 6. C# 12 Yenilikleri

.NET 9, C# 12 ile birlikte geliyor. Bu sürüm, programlama deneyimini daha modern ve güçlü hale getiren birçok yenilik
sunar.

- **Primary Constructors**

```csharp
public class Person(string Name, int Age)
{
    public void PrintInfo() => Console.WriteLine($"{Name}, {Age} yaşında.");
}
var person = new Person("Ali", 30);
person.PrintInfo();
```

- **Lambda Enhancements**

```csharp
var square = (int x) => x * x;
Console.WriteLine(square(5)); // Çıktı: 25
```

---

## 7. SIMD ve Vectorization Geliştirmeleri

.NET 9, SIMD (Single Instruction Multiple Data) ve vektörleştirme desteği ile performans açısından önemli gelişmeler sunar. Bu özellikler, aynı anda birden fazla veri elemanını işleyerek özellikle büyük veri setleriyle çalışan uygulamalarda büyük bir hız artışı sağlar.SIMD ve vektörleştirme, özellikle büyük veri işleme ve hesaplama ağırlıklı uygulamalar için güçlü bir araçtır. Bu özellikler, .NET 9’un performans odaklı yeniliklerinden biridir ve uygun şekilde kullanıldığında uygulamanızın hızını önemli ölçüde artırabilir.

---

### Avantajları
1. **Paralel İşleme:** Birden fazla veri elemanını aynı anda işleyerek işlem sürelerini kısaltır.
2. **Hesaplama Performansı:** Matematiksel ve veri manipülasyonu işlemlerinde daha hızlı sonuçlar.
3. **Düşük Bellek Kullanımı:** Verimli veri işleme sayesinde bellek kullanımı optimize edilir.

---

### Temel Kullanım

Aşağıdaki örnekte, iki vektörün eleman bazında toplandığı bir senaryo gösterilmektedir:

```csharp
using System.Numerics;

public class Program
{
    public static void Main()
    {
        Vector<int> vector1 = new Vector<int>(new int[] { 1, 2, 3, 4 });
        Vector<int> vector2 = new Vector<int>(new int[] { 5, 6, 7, 8 });

        Vector<int> result = vector1 + vector2;

        Console.WriteLine(result); // Çıktı: <6, 8, 10, 12>
    }
}
```

---

### Büyük Veri İşlemleri

SIMD, büyük veri setlerini hızla işlemek için idealdir. Aşağıdaki örnek, bir dizideki elemanların karesini hesaplar:

```csharp
using System.Numerics;

public class Program
{
    public static void Main()
    {
        int[] data = new int[1000];
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = i;
        }

        int vectorSize = Vector<int>.Count;
        for (int i = 0; i < data.Length; i += vectorSize)
        {
            Vector<int> vector = new Vector<int>(data, i);
            Vector<int> squared = vector * vector;
            squared.CopyTo(data, i);
        }

        Console.WriteLine($"İlk eleman: {data[0]}, Son eleman: {data[999]}");
    }
}
```

Bu kod, verileri vektörler halinde işleyerek performansı artırır.

---

### SIMD ile Matris Çarpımı

Matris çarpımı gibi yoğun hesaplamalı işlemler SIMD ile önemli ölçüde hızlandırılabilir.

```csharp
using System.Numerics;

public static class MatrixMultiplier
{
    public static void Multiply(float[,] matrixA, float[,] matrixB, float[,] result)
    {
        int rows = matrixA.GetLength(0);
        int cols = matrixB.GetLength(1);

        for (int i = 0; i < rows; i++)
        {
            for (int j = 0; j < cols; j++)
            {
                float sum = 0;
                for (int k = 0; k < Vector<float>.Count; k++)
                {
                    Vector<float> rowVector = new Vector<float>(matrixA, i * cols + k);
                    Vector<float> colVector = new Vector<float>(matrixB, k * cols + j);
                    sum += Vector.Dot(rowVector, colVector);
                }
                result[i, j] = sum;
            }
        }
    }
}
```

Bu yaklaşım, matris çarpımlarını paralel işleyerek yoğun hesaplama yükünü hafifletir.

---

### Performans Analizi

Aynı veri setini SIMD ile ve olmadan işlemek için basit bir karşılaştırma:

#### SIMD Kullanılmadan:
```csharp
for (int i = 0; i < data.Length; i++)
{
    data[i] = data[i] * data[i];
}
```

#### SIMD ile:
```csharp
for (int i = 0; i < data.Length; i += vectorSize)
{
    Vector<int> vector = new Vector<int>(data, i);
    Vector<int> squared = vector * vector;
    squared.CopyTo(data, i);
}
```

Sonuçlar genellikle SIMD ile 2-5 kat arasında hızlanma sağlar.

---

### Sınırlamalar ve Dikkat Edilmesi Gerekenler

1. **Donanım Desteği:** SIMD işlemleri, CPU’nuzun SIMD desteğine bağlıdır.
2. **Veri Boyutu:** SIMD, sabit boyutlu vektörlerde en verimlidir.
3. **Debug Zorluğu:** Paralel işlemleri debug etmek daha zor olabilir.

---

## Özet

.NET 9, geliştirme süreçlerini kolaylaştıran ve modern yazılım geliştirme ihtiyaçlarını karşılayan birçok yenilik
sunuyor. Bu özellikler, hem performans hem de kullanım kolaylığı açısından geliştiricilere önemli avantajlar sağlar.

---

## İlgili Kaynaklar

- [Microsoft Docs: .NET 9](https://learn.microsoft.com/en-us/dotnet/)
- [HTTP/3 Documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/http3)
- [Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview)