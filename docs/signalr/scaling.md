# SignalR Scaling

SignalR ölçeklenmesi, birden fazla sunucuda gerçek zamanlı iletişimin tutarlı çalışmasını sağlar; yanlış yapılandırma mesaj kaybına ve bağlantı sorunlarına yol açar.

---

## 1. Backplane Olmadan Çoklu Sunucu

❌ **Yanlış Kullanım:** Birden fazla sunucuda backplane kullanmamak.

```csharp
builder.Services.AddSignalR(); // Tek sunucuda çalışır
// Load balancer arkasında Server1'deki kullanıcı Server2'dekine mesaj gönderemez
```

✅ **İdeal Kullanım:** Redis backplane ile sunucular arası mesajlaşma sağlayın.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(builder.Configuration.GetConnectionString("Redis"), options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("MyApp");
    });
```

---

## 2. Sticky Sessions Yönetmemek

❌ **Yanlış Kullanım:** WebSocket bağlantısını load balancer'da dengelememek.

```csharp
// Load balancer her isteği farklı sunucuya yönlendirir
// WebSocket handshake ve bağlantı farklı sunuculara düşer - başarısız
```

✅ **İdeal Kullanım:** Sticky session veya WebSocket desteği olan load balancer kullanın.

```csharp
// Azure SignalR Service ile ölçekleme sorununu tamamen ortadan kaldırın
builder.Services.AddSignalR()
    .AddAzureSignalR(builder.Configuration["Azure:SignalR:ConnectionString"]);

// Veya Nginx sticky session
// upstream backend {
//     ip_hash;
//     server app1:5000;
//     server app2:5000;
// }
```

---

## 3. MessagePack Kullanmamak

❌ **Yanlış Kullanım:** Büyük veri transferinde JSON kullanmak.

```csharp
builder.Services.AddSignalR(); // Varsayılan JSON protokolü
// Büyük mesajlarda yüksek bandwidth ve yavaş serialization
```

✅ **İdeal Kullanım:** MessagePack ile daha hızlı ve küçük mesajlar gönderin.

```csharp
builder.Services.AddSignalR()
    .AddMessagePackProtocol();

// İstemci tarafı
var connection = new HubConnectionBuilder()
    .WithUrl("/chatHub")
    .AddMessagePackProtocol()
    .Build();
```

---

## 4. Reconnection Stratejisi Olmamak

❌ **Yanlış Kullanım:** Bağlantı koptuğunda yeniden bağlanma yapmamak.

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("/chatHub")
    .Build();

await connection.StartAsync();
// Bağlantı koparsa kullanıcı kalıcı olarak bağlantısını kaybeder
```

✅ **İdeal Kullanım:** Otomatik yeniden bağlanma stratejisi tanımlayın.

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("/chatHub")
    .WithAutomaticReconnect(new[] { TimeSpan.Zero, TimeSpan.FromSeconds(2),
        TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(10) })
    .Build();

connection.Reconnecting += error =>
{
    // UI'da "Yeniden bağlanılıyor..." göster
    return Task.CompletedTask;
};

connection.Reconnected += connectionId =>
{
    // Kaçırılan mesajları yeniden yükle
    return Task.CompletedTask;
};

connection.Closed += async error =>
{
    await Task.Delay(5000);
    await connection.StartAsync(); // Son çare: manuel yeniden başlat
};
```

---

## 5. Authentication Yapmamak

❌ **Yanlış Kullanım:** Hub'a kimlik doğrulaması olmadan erişim izni vermek.

```csharp
public class AdminHub : Hub
{
    public async Task DeleteUser(int userId)
    {
        await _userService.DeleteAsync(userId); // Herkes admin işlemi yapabilir!
    }
}
```

✅ **İdeal Kullanım:** Hub'a authentication ve authorization uygulayın.

```csharp
[Authorize]
public class AdminHub : Hub
{
    [Authorize(Roles = "Admin")]
    public async Task DeleteUser(int userId)
    {
        await _userService.DeleteAsync(userId);
        await Clients.All.SendAsync("UserDeleted", userId);
    }
}

// JWT ile SignalR authentication
builder.Services.AddAuthentication().AddJwtBearer(options =>
{
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = context =>
        {
            var accessToken = context.Request.Query["access_token"];
            var path = context.HttpContext.Request.Path;
            if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
            {
                context.Token = accessToken;
            }
            return Task.CompletedTask;
        }
    };
});
```