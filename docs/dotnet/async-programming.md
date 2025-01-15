# Asenkron Programlama ve Task Yönetimi

Asenkron programlama, modern uygulamalarda performans ve yanıt verebilirlik açısından büyük önem taşır. Ancak, yanlış kullanıldığında beklenmedik sorunlara yol açabilir. İşte C#'ta asenkron programlama için sık yapılan hatalar ve önerilen çözümler.

---

## 1. `await` Kullanımını Atlamak

❌ **Yanlış Kullanım:** `await` kullanılmadığında istisna durumları doğru şekilde ele alınamaz.

```csharp
try
{
    DoWorkWithoutAwaitAsync();
}
catch (Exception e)
{
    Console.WriteLine($"Hata: {e.Message}");
}

static Task DoWorkWithoutAwaitAsync()
{
    return ThrowExceptionAsync();
}

static async Task ThrowExceptionAsync()
{
    await Task.Yield();
    throw new Exception("Bir hata oluştu!");
}
```

✅ **İdeal Kullanım:** `await` ile çağrıyı bekleyerek hata yönetimini iyileştirin.

```csharp
try
{
    await DoWorkWithAwaitAsync();
}
catch (Exception e)
{
    Console.WriteLine($"Hata: {e.Message}");
}

static async Task DoWorkWithAwaitAsync()
{
    await ThrowExceptionAsync();
}

static async Task ThrowExceptionAsync()
{
    await Task.Yield();
    throw new Exception("Bir hata oluştu!");
}
```

---

## 2. `async void` Kullanımı

❌ **Yanlış Kullanım:** `async void` hataları doğru bir şekilde ele almayı zorlaştırır.

```csharp
public async void DoAsync()
{
    await SomeAsyncOperation();
}
```

✅ **İdeal Kullanım:** `async Task` kullanarak hata yönetimini ve test edilebilirliği artırın.

```csharp
public async Task DoAsync()
{
    await SomeAsyncOperation();
}
```

> 💡 **Not:** `async void` yalnızca olay işleyicileri gibi özel durumlarda kullanılmalıdır.

---

## 3. `Task` Nesnesini Beklemeden Döndürmek

❌ **Yanlış Kullanım:** `using` bloğu içinde `Task` nesnesini beklemeden döndürmek kaynakların erken serbest bırakılmasına neden olabilir.

```csharp
public Task<string> GetContentAsync()
{
    using var client = new HttpClient();
    return client.GetStringAsync("https://example.com");
}
```

✅ **İdeal Kullanım:** `await` kullanarak kaynak yönetimini doğru yapın.

```csharp
public async Task<string> GetContentAsync()
{
    using var client = new HttpClient();
    return await client.GetStringAsync("https://example.com");
}
```

---

## 4. Paralel İşlemlerin Yanlış Yönetimi

❌ **Yanlış Kullanım:** Paralel görevleri sırayla beklemek.

```csharp
await Task1();
await Task2();
```

✅ **İdeal Kullanım:** Görevleri paralel olarak başlatıp aynı anda beklemek.

```csharp
var task1 = Task1();
var task2 = Task2();
await Task.WhenAll(task1, task2);
```

---

## 5. `Task.Delay` ile Hassas Bekleme

❌ **Yanlış Kullanım:** Kısa süreli hassas bekleme için `Task.Delay` kullanmak.

```csharp
await Task.Delay(1); // Süre tam olarak 1ms olmayabilir.
```

✅ **İdeal Kullanım:** Hassas zamanlama için daha uygun araçlar kullanın.

---

## 6. Deadlock Sorunları

❌ **Yanlış Kullanım:** `Task.Result` veya `Task.Wait` kullanarak deadlock oluşturmak.

```csharp
var result = SomeAsyncOperation().Result;
```

✅ **İdeal Kullanım:** `await` kullanarak deadlock sorunlarını önleyin.

```csharp
var result = await SomeAsyncOperation();
```

---

## 7. `Task` Nesnelerini Geri Dönüştürmek

❌ **Yanlış Kullanım:** Her işlem için yeni bir `Task` nesnesi oluşturmak.

```csharp
public Task<int> GetNumberAsync()
{
    return Task.Run(() => 42);
}
```

✅ **İdeal Kullanım:** `Task.FromResult` kullanarak gereksiz nesne oluşturmayı önleyin.

```csharp
public Task<int> GetNumberAsync()
{
    return Task.FromResult(42);
}
```