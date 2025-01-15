# Default Interface Methods

Default interface methods, C# 8.0 ile gelen bir özellik olup, arabirimlere varsayılan metot tanımlamaları eklemenizi sağlar. Bu, arabirimlerin geriye dönük uyumluluğunu artırır, ancak yanlış kullanımları kodun karmaşık hale gelmesine ve bakımı zorlaşmasına neden olabilir.

---

## 1. Default Method'ları Gereksiz Yere Kullanmak

❌ **Yanlış Kullanım:** Default method'ları temel sınıf işlevsellikleri için kullanmak.

```csharp
public interface ILogger
{
    void Log(string message);

    void LogError(string message) // Gereksiz bir default method
    {
        Console.WriteLine($"Error: {message}");
    }
}
```

✅ **İdeal Kullanım:** Default method'ları yalnızca geriye dönük uyumluluk için kullanın.

```csharp
public interface ILogger
{
    void Log(string message);

    void LogError(string message)
    {
        Log($"Error: {message}");
    }
}
```

---

## 2. Karmaşık İş Mantığı Eklemek

❌ **Yanlış Kullanım:** Default method'larda karmaşık iş mantığı tanımlamak.

```csharp
public interface IDataProcessor
{
    void ProcessData(string data);

    void ValidateData(string data)
    {
        if (string.IsNullOrEmpty(data))
        {
            throw new ArgumentException("Data is required.");
        }
        // Daha karmaşık işlemler...
    }
}
```

✅ **İdeal Kullanım:** Karmaşık mantığı sınıflarda veya ayrı hizmetlerde ele alın.

```csharp
public interface IDataProcessor
{
    void ProcessData(string data);

    void ValidateData(string data)
    {
        if (string.IsNullOrEmpty(data))
        {
            throw new ArgumentException("Data is required.");
        }
    }
}
```

---

## 3. Varsayılan Yöntemleri Sıkça Kullanarak Arabirimi Şişirmek

❌ **Yanlış Kullanım:** Arabirimlere aşırı sayıda varsayılan metot eklemek.

```csharp
public interface IReportGenerator
{
    void GenerateReport();
    void ExportReport(string format) { Console.WriteLine($"Exporting in {format} format."); }
    void PrintReport() { Console.WriteLine("Printing report."); }
}
```

✅ **İdeal Kullanım:** Arabirimi sade tutun ve gerektiğinde yeni arabirimler oluşturun.

```csharp
public interface IReportGenerator
{
    void GenerateReport();
    void ExportReport(string format)
    {
        Console.WriteLine($"Exporting in {format} format.");
    }
}
```

---

## 4. Default Method'ları Tüm Sınıflarda Kullanmamak

❌ **Yanlış Kullanım:** Tüm arabirim implementasyonlarında default method'u göz ardı etmek.

```csharp
public interface INotifier
{
    void Notify(string message);

    void NotifyAll(string[] messages)
    {
        foreach (var message in messages)
        {
            Notify(message);
        }
    }
}
```

✅ **İdeal Kullanım:** Default method'un avantajlarını tüm implementasyonlarda kullanın.

```csharp
public interface INotifier
{
    void Notify(string message);

    void NotifyAll(string[] messages)
    {
        foreach (var message in messages)
        {
            Notify(message);
        }
    }
}
```

---

## 5. Test Edilebilirliği Göz Ardı Etmek

❌ **Yanlış Kullanım:** Default method'ları test edilebilirlik açısından zorlaştırmak.

```csharp
public interface IService
{
    void PerformAction();

    void DefaultAction()
    {
        Console.WriteLine("Default action performed.");
    }
}
```

✅ **İdeal Kullanım:** Default method'ları bir temel sınıf veya bağımlılık üzerinden test edilebilir hale getirin.

```csharp
public interface IService
{
    void PerformAction();

    void DefaultAction()
    {
        PerformAction();
    }
}
```