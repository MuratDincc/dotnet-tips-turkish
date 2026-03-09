# Builder Pattern

Builder deseni, karmaşık nesnelerin adım adım oluşturulmasını sağlar; yanlış kullanımlar ise uzun constructor parametreleri ve tutarsız nesne durumlarına yol açar.

---

## 1. Telescoping Constructor Anti-Pattern

❌ **Yanlış Kullanım:** Çok sayıda parametreye sahip constructor kullanmak.

```csharp
var email = new Email(
    "alici@test.com",
    "gonderen@test.com",
    "Konu",
    "İçerik",
    true,
    null,
    "UTF-8",
    Priority.High,
    null,
    true
); // Parametrelerin anlamı belirsiz
```

✅ **İdeal Kullanım:** Builder ile okunabilir nesne oluşturma sağlayın.

```csharp
var email = new EmailBuilder()
    .To("alici@test.com")
    .From("gonderen@test.com")
    .Subject("Konu")
    .Body("İçerik")
    .IsHtml(true)
    .WithPriority(Priority.High)
    .WithReadReceipt(true)
    .Build();
```

---

## 2. Fluent API'de Immutability Sağlamamak

❌ **Yanlış Kullanım:** Builder'ın mutable state ile çalışması.

```csharp
var builder = new QueryBuilder();
builder.Select("Name");
builder.Where("Age > 18");

var query1 = builder.Build(); // SELECT Name WHERE Age > 18

builder.Where("City = 'Istanbul'");
var query2 = builder.Build(); // SELECT Name WHERE Age > 18 AND City = 'Istanbul'
// query1 etkilenmez ama builder state'i paylaşılıyorsa sorun çıkar
```

✅ **İdeal Kullanım:** Her adımda yeni bir builder instance döndürün.

```csharp
public class QueryBuilder
{
    private readonly List<string> _columns;
    private readonly List<string> _conditions;

    private QueryBuilder(List<string> columns, List<string> conditions)
    {
        _columns = columns;
        _conditions = conditions;
    }

    public QueryBuilder() : this(new(), new()) { }

    public QueryBuilder Select(string column)
        => new QueryBuilder(new(_columns) { column }, new(_conditions));

    public QueryBuilder Where(string condition)
        => new QueryBuilder(new(_columns), new(_conditions) { condition });

    public string Build()
        => $"SELECT {string.Join(", ", _columns)} WHERE {string.Join(" AND ", _conditions)}";
}
```

---

## 3. Zorunlu Alanları Kontrol Etmemek

❌ **Yanlış Kullanım:** Build sırasında validasyon yapmamak.

```csharp
var config = new AppConfigBuilder()
    .WithTimeout(30)
    .Build(); // ConnectionString olmadan geçersiz config oluşur
```

✅ **İdeal Kullanım:** Build sırasında zorunlu alanları doğrulayın.

```csharp
public class AppConfigBuilder
{
    private string _connectionString;
    private int _timeout = 30;

    public AppConfigBuilder WithConnectionString(string cs)
    {
        _connectionString = cs;
        return this;
    }

    public AppConfigBuilder WithTimeout(int timeout)
    {
        _timeout = timeout;
        return this;
    }

    public AppConfig Build()
    {
        if (string.IsNullOrEmpty(_connectionString))
            throw new InvalidOperationException("ConnectionString zorunludur.");

        return new AppConfig(_connectionString, _timeout);
    }
}
```

---

## 4. Builder Yerine Object Initializer Kullanılabilecek Yerde Builder Yazmak

❌ **Yanlış Kullanım:** Basit nesneler için gereksiz builder oluşturmak.

```csharp
public class AddressBuilder
{
    private string _street, _city, _zip;

    public AddressBuilder WithStreet(string s) { _street = s; return this; }
    public AddressBuilder WithCity(string c) { _city = c; return this; }
    public AddressBuilder WithZip(string z) { _zip = z; return this; }
    public Address Build() => new Address(_street, _city, _zip);
}

var address = new AddressBuilder()
    .WithStreet("Atatürk Cad.")
    .WithCity("İstanbul")
    .WithZip("34000")
    .Build();
```

✅ **İdeal Kullanım:** Basit nesnelerde object initializer veya record kullanın.

```csharp
public record Address(string Street, string City, string Zip);

var address = new Address("Atatürk Cad.", "İstanbul", "34000");

// veya init-only properties ile
public class Address
{
    public required string Street { get; init; }
    public required string City { get; init; }
    public required string Zip { get; init; }
}

var address = new Address { Street = "Atatürk Cad.", City = "İstanbul", Zip = "34000" };
```

---

## 5. Step Builder ile Zorunlu Adımları Garanti Etmek

❌ **Yanlış Kullanım:** Tüm adımları isteğe bağlı bırakmak.

```csharp
var request = new HttpRequestBuilder()
    .WithBody("data")
    .Build(); // URL ve Method belirtilmedi, runtime hatası
```

✅ **İdeal Kullanım:** Step Builder ile derleme zamanında zorunlu adımları garanti edin.

```csharp
public interface ISetUrl
{
    ISetMethod WithUrl(string url);
}

public interface ISetMethod
{
    IBuildable WithMethod(HttpMethod method);
}

public interface IBuildable
{
    IBuildable WithBody(string body);
    HttpRequestMessage Build();
}

public class HttpRequestBuilder : ISetUrl, ISetMethod, IBuildable
{
    private string _url;
    private HttpMethod _method;
    private string _body;

    public static ISetUrl Create() => new HttpRequestBuilder();

    public ISetMethod WithUrl(string url) { _url = url; return this; }
    public IBuildable WithMethod(HttpMethod method) { _method = method; return this; }
    public IBuildable WithBody(string body) { _body = body; return this; }

    public HttpRequestMessage Build()
    {
        var request = new HttpRequestMessage(_method, _url);
        if (_body != null) request.Content = new StringContent(_body);
        return request;
    }
}

// Kullanım: URL ve Method olmadan Build çağrılamaz
var request = HttpRequestBuilder.Create()
    .WithUrl("https://api.example.com")
    .WithMethod(HttpMethod.Post)
    .WithBody("{\"name\":\"test\"}")
    .Build();
```