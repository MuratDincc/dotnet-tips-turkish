# Hub Design

SignalR Hub, istemci-sunucu arası gerçek zamanlı iletişimin merkezidir; yanlış tasarım bellek sızıntılarına ve ölçeklenme sorunlarına yol açar.

---

## 1. Hub İçinde İş Mantığı Yazmak

❌ **Yanlış Kullanım:** Hub metodunda veritabanı erişimi ve iş mantığı yazmak.

```csharp
public class ChatHub : Hub
{
    private readonly AppDbContext _context;

    public async Task SendMessage(string message)
    {
        var user = await _context.Users.FindAsync(Context.UserIdentifier);
        if (user.IsBanned) return;

        var chatMessage = new ChatMessage { UserId = user.Id, Content = message };
        _context.Messages.Add(chatMessage);
        await _context.SaveChangesAsync();

        await Clients.All.SendAsync("ReceiveMessage", user.Name, message);
    }
}
```

✅ **İdeal Kullanım:** Hub'ı ince tutun, iş mantığını servise taşıyın.

```csharp
public class ChatHub : Hub
{
    private readonly IChatService _chatService;

    public ChatHub(IChatService chatService) => _chatService = chatService;

    public async Task SendMessage(string message)
    {
        var result = await _chatService.SendMessageAsync(Context.UserIdentifier, message);
        if (result.IsSuccess)
        {
            await Clients.All.SendAsync("ReceiveMessage", result.SenderName, message);
        }
    }
}
```

---

## 2. Connection Lifecycle Yönetmemek

❌ **Yanlış Kullanım:** Bağlantı açılıp kapandığında temizlik yapmamak.

```csharp
public class GameHub : Hub
{
    private static readonly Dictionary<string, string> _onlineUsers = new();

    public async Task JoinGame(string username)
    {
        _onlineUsers[Context.ConnectionId] = username;
        await Clients.All.SendAsync("UserJoined", username);
    }
    // Kullanıcı bağlantıyı kapatırsa _onlineUsers'dan silinmez
}
```

✅ **İdeal Kullanım:** OnConnectedAsync ve OnDisconnectedAsync ile lifecycle yönetin.

```csharp
public class GameHub : Hub
{
    private readonly IConnectionTracker _tracker;

    public GameHub(IConnectionTracker tracker) => _tracker = tracker;

    public override async Task OnConnectedAsync()
    {
        await _tracker.AddConnectionAsync(Context.ConnectionId, Context.UserIdentifier);
        await base.OnConnectedAsync();
    }

    public override async Task OnDisconnectedAsync(Exception exception)
    {
        var username = await _tracker.RemoveConnectionAsync(Context.ConnectionId);
        await Clients.All.SendAsync("UserLeft", username);
        await base.OnDisconnectedAsync(exception);
    }
}
```

---

## 3. Group Yönetimini Yanlış Yapmak

❌ **Yanlış Kullanım:** Tüm istemcilere mesaj göndermek.

```csharp
public async Task SendRoomMessage(string roomId, string message)
{
    await Clients.All.SendAsync("ReceiveMessage", roomId, message);
    // Tüm kullanıcılar tüm odaların mesajlarını alır
}
```

✅ **İdeal Kullanım:** Group'lar ile sadece ilgili kullanıcılara gönderin.

```csharp
public async Task JoinRoom(string roomId)
{
    await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
    await Clients.Group(roomId).SendAsync("UserJoined", Context.UserIdentifier);
}

public async Task LeaveRoom(string roomId)
{
    await Groups.RemoveFromGroupAsync(Context.ConnectionId, roomId);
    await Clients.Group(roomId).SendAsync("UserLeft", Context.UserIdentifier);
}

public async Task SendRoomMessage(string roomId, string message)
{
    await Clients.Group(roomId).SendAsync("ReceiveMessage", Context.UserIdentifier, message);
}
```

---

## 4. Strongly-Typed Hub Kullanmamak

❌ **Yanlış Kullanım:** Magic string ile metod çağrısı yapmak.

```csharp
await Clients.All.SendAsync("ReceiveMessage", user, message);
await Clients.All.SendAsync("RecieveMessage", user, message); // Typo, runtime'da hata
```

✅ **İdeal Kullanım:** Strongly-typed hub interface tanımlayın.

```csharp
public interface IChatClient
{
    Task ReceiveMessage(string user, string message);
    Task UserJoined(string user);
    Task UserLeft(string user);
}

public class ChatHub : Hub<IChatClient>
{
    public async Task SendMessage(string message)
    {
        await Clients.All.ReceiveMessage(Context.UserIdentifier, message);
    }

    public async Task JoinRoom(string roomId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, roomId);
        await Clients.Group(roomId).UserJoined(Context.UserIdentifier);
    }
}
```

---

## 5. Hub Dışından Mesaj Gönderememek

❌ **Yanlış Kullanım:** Hub instance'ını oluşturup mesaj göndermeye çalışmak.

```csharp
public class OrderService
{
    public async Task CompleteOrderAsync(int orderId)
    {
        // Hub'a nasıl erişilir?
        var hub = new NotificationHub(); // Çalışmaz!
    }
}
```

✅ **İdeal Kullanım:** IHubContext ile Hub dışından mesaj gönderin.

```csharp
public class OrderService
{
    private readonly IHubContext<NotificationHub, INotificationClient> _hubContext;

    public OrderService(IHubContext<NotificationHub, INotificationClient> hubContext)
        => _hubContext = hubContext;

    public async Task CompleteOrderAsync(int orderId, string customerId)
    {
        await _hubContext.Clients.User(customerId)
            .OrderCompleted(orderId, "Siparişiniz tamamlandı!");
    }
}
```