# gRPC Service Design

gRPC, yüksek performanslı servisler arası iletişim sağlar; yanlış tasarım geriye uyumluluk sorunlarına ve performans kayıplarına yol açar.

---

## 1. Proto Dosyasını Yanlış Tasarlamak

❌ **Yanlış Kullanım:** Büyük ve monolitik proto dosyası oluşturmak.

```protobuf
syntax = "proto3";

service AppService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
    rpc GetProduct (GetProductRequest) returns (ProductResponse);
    rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
    // Tek servis, tüm domain'ler karışık
}
```

✅ **İdeal Kullanım:** Domain bazlı ayrı proto dosyaları oluşturun.

```protobuf
// user.proto
syntax = "proto3";
package myapp.users.v1;

service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
}

// order.proto
syntax = "proto3";
package myapp.orders.v1;

service OrderService {
    rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
    rpc GetOrder (GetOrderRequest) returns (OrderResponse);
}
```

---

## 2. Field Numaralarını Değiştirmek

❌ **Yanlış Kullanım:** Mevcut field numaralarını yeniden kullanmak veya değiştirmek.

```protobuf
message Product {
    int32 id = 1;
    // string name = 2; silindi
    string title = 2;  // Aynı numara farklı field - breaking change!
    decimal price = 3;
}
```

✅ **İdeal Kullanım:** Eski numaraları reserved olarak işaretleyin, yeni field'a yeni numara verin.

```protobuf
message Product {
    int32 id = 1;
    reserved 2; // Eski "name" field'ı
    reserved "name";
    string title = 4; // Yeni field, yeni numara
    double price = 3;
}
```

---

## 3. Error Handling Yapmamak

❌ **Yanlış Kullanım:** Exception'ı doğrudan fırlatmak.

```csharp
public override async Task<OrderResponse> GetOrder(GetOrderRequest request, ServerCallContext context)
{
    var order = await _repository.GetByIdAsync(request.Id);
    if (order == null) throw new Exception("Sipariş bulunamadı"); // Generic exception
    return MapToResponse(order);
}
```

✅ **İdeal Kullanım:** gRPC status codes ile yapılandırılmış hata döndürün.

```csharp
public override async Task<OrderResponse> GetOrder(GetOrderRequest request, ServerCallContext context)
{
    var order = await _repository.GetByIdAsync(request.Id);

    if (order == null)
    {
        throw new RpcException(new Status(StatusCode.NotFound,
            $"Sipariş bulunamadı: {request.Id}"));
    }

    return MapToResponse(order);
}

// Global interceptor ile hata yönetimi
public class ErrorInterceptor : Interceptor
{
    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request, ServerCallContext context, UnaryServerMethod<TRequest, TResponse> continuation)
    {
        try
        {
            return await continuation(request, context);
        }
        catch (RpcException) { throw; }
        catch (Exception ex)
        {
            _logger.LogError(ex, "gRPC hatası");
            throw new RpcException(new Status(StatusCode.Internal, "Sunucu hatası"));
        }
    }
}
```

---

## 4. Deadline Belirlememek

❌ **Yanlış Kullanım:** Timeout olmadan servis çağrısı yapmak.

```csharp
var response = await _client.GetOrderAsync(new GetOrderRequest { Id = 1 });
// Servis yanıt vermezse sonsuza kadar bekler
```

✅ **İdeal Kullanım:** Deadline ile timeout belirleyin.

```csharp
var deadline = DateTime.UtcNow.AddSeconds(5);
var response = await _client.GetOrderAsync(
    new GetOrderRequest { Id = 1 },
    deadline: deadline);

// Server tarafında deadline kontrolü
public override async Task<OrderResponse> GetOrder(GetOrderRequest request, ServerCallContext context)
{
    if (context.Deadline < DateTime.UtcNow)
    {
        throw new RpcException(new Status(StatusCode.DeadlineExceeded, "Süre aşıldı"));
    }

    return await ProcessAsync(request, context.CancellationToken);
}
```

---

## 5. Streaming Kullanmamak

❌ **Yanlış Kullanım:** Büyük veri setini tek response'da döndürmek.

```csharp
public override async Task<ProductListResponse> GetAllProducts(
    Empty request, ServerCallContext context)
{
    var products = await _repository.GetAllAsync(); // 100.000 kayıt, bellek sorunu
    return new ProductListResponse { Products = { products.Select(MapToProto) } };
}
```

✅ **İdeal Kullanım:** Server streaming ile büyük veriyi parça parça gönderin.

```csharp
public override async Task GetAllProducts(
    Empty request,
    IServerStreamWriter<ProductResponse> responseStream,
    ServerCallContext context)
{
    await foreach (var product in _repository.GetAllAsyncStream(context.CancellationToken))
    {
        await responseStream.WriteAsync(MapToProto(product));
    }
}

// Client tarafı
using var call = _client.GetAllProducts(new Empty());
await foreach (var product in call.ResponseStream.ReadAllAsync())
{
    Console.WriteLine(product.Name);
}
```