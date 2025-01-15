# OpenAPI ile API Dokümantasyonu

OpenAPI, RESTful API'lerin nasıl çalıştığını açıklamak için kullanılan standart bir spesifikasyondur. .NET 9, API'lerinizi kolayca belgelemek ve test etmek için OpenAPI desteği sunar. Ancak, bu sürümde Swagger'ın desteği sona ermiştir. Bunun yerine modern OpenAPI yaklaşımı benimsenmiştir.

---

## Swagger'ın Kaldırılması

.NET 9 ile birlikte Swagger desteği resmi olarak kaldırılmıştır. Bu konu hakkında daha fazla bilgi edinmek için aşağıdaki makaleyi okuyabilirsiniz:

[Swagger'a Elveda: .NET 9 ile API Dokümantasyonu OpenAPI](https://medium.com/devopsturkiye/swaggera-elveda-net-9-ile-api-dok%C3%BCmantasyonu-openapi-8c06fb2f7210)

Bu makale, Swagger'dan OpenAPI'ye geçiş sürecini ve bunun avantajlarını detaylı bir şekilde ele alır.

---

## OpenAPI Entegrasyonu

Bir ASP.NET Core projesine OpenAPI eklemek için şu adımları izleyebilirsiniz:

```csharp
var builder = WebApplication.CreateBuilder(args);

// OpenAPI hizmetlerini ekleyin
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Swagger Middleware'ini ekleyin
app.UseSwagger();
app.UseSwaggerUI();

app.MapGet("/", () => "Merhaba, API!");
app.Run();
```

---

## Geliştirme Ortamında OpenAPI Kullanımı

OpenAPI arayüzünü yalnızca geliştirme ortamında etkinleştirmek için şu kontrolü ekleyin:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

---

## OpenAPI Parametreleri ve Kullanımı

OpenAPI, bir API'nin nasıl çalıştığını açıklamak için belirli parametreleri kullanır. Bu parametreler, API'nin giriş ve çıkışlarını anlamak için kritik öneme sahiptir.

### 1. `paths`

- **Açıklama:** API endpoint'lerini tanımlar. Her bir endpoint için HTTP metodları (GET, POST, PUT vb.) ve bunların detayları burada yer alır.
- **Kullanım Örneği:**
```yaml
paths:
  /users:
    get:
      summary: Kullanıcı listesini döndürür.
      responses:
        '200':
          description: Başarılı.
```

### 2. `parameters`

- **Açıklama:** API'nin istekte beklediği parametreleri tanımlar. Parametreler genellikle URL'de, sorgu dizisinde veya başlıklarda bulunur.
- **Kullanım Örneği:**
```yaml
parameters:
  - name: id
    in: query
    description: Kullanıcının benzersiz kimliği.
    required: true
    schema:
      type: integer
```

### 3. `requestBody`

- **Açıklama:** POST veya PUT gibi metodlar için gönderilen veri gövdesini tanımlar.
- **Kullanım Örneği:**
```yaml
requestBody:
  description: Yeni bir kullanıcı oluşturmak için veri.
  required: true
  content:
    application/json:
      schema:
        type: object
        properties:
          name:
            type: string
          age:
            type: integer
```

### 4. `responses`

- **Açıklama:** API'nin döndürebileceği HTTP durum kodları ve ilgili açıklamaları tanımlar.
- **Kullanım Örneği:**
```yaml
responses:
  '200':
    description: Başarılı.
    content:
      application/json:
        schema:
          type: object
          properties:
            id:
              type: integer
            name:
              type: string
```

### 5. `security`

- **Açıklama:** API'nin güvenlik gereksinimlerini belirtir (örneğin, Bearer Token veya API Key).
- **Kullanım Örneği:**
```yaml
security:
  - bearerAuth: []
```

---

## İlgili Kaynaklar

- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/tutorials/getting-started-with-swashbuckle)
- [Swagger UI](https://swagger.io/tools/swagger-ui/)