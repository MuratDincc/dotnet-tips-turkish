# Deployment Strategies

Deployment stratejileri, uygulamanın kesintisiz güncellenmesini sağlar; yanlış strateji seçimi downtime ve kullanıcı kaybına yol açar.

---

## 1. Big Bang Deployment

❌ **Yanlış Kullanım:** Tüm instance'ları aynı anda güncellemek.

```yaml
# Tüm pod'lar aynı anda öldürülüp yenisi başlatılır
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: Recreate  # Downtime kaçınılmaz
  replicas: 3
```

✅ **İdeal Kullanım:** Rolling update ile kademeli güncelleme yapın.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:v2
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## 2. Readiness Probe Olmadan Deploy

❌ **Yanlış Kullanım:** Uygulama hazır olmadan trafik yönlendirmek.

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:v2
      ports:
        - containerPort: 8080
      # Probe yok, container başlar başlamaz trafik alır
      # Uygulama henüz warm-up tamamlamamış olabilir
```

✅ **İdeal Kullanım:** Liveness ve readiness probe'ları tanımlayın.

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:v2
      ports:
        - containerPort: 8080
      startupProbe:
        httpGet:
          path: /health/startup
          port: 8080
        failureThreshold: 30
        periodSeconds: 2
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 3
      livenessProbe:
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 3
```

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("startup", () => _isReady ? HealthCheckResult.Healthy() : HealthCheckResult.Unhealthy())
    .AddDbContextCheck<AppDbContext>();

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
app.MapHealthChecks("/health/live");
```

---

## 3. Blue-Green Deployment Yapmamak

❌ **Yanlış Kullanım:** In-place update ile risk almak.

```bash
# Mevcut uygulama durdurulur, yenisi başlatılır
systemctl stop myapp
cp -r /new-version/* /app/
systemctl start myapp
# Hata varsa geri dönüş zor
```

✅ **İdeal Kullanım:** Blue-Green deployment ile anında geçiş yapın.

```yaml
# Blue (mevcut) ve Green (yeni) environment'lar paralel çalışır
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: green  # Trafik green'e yönlendirilir
  ports:
    - port: 80
      targetPort: 8080

---
# Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
```

```bash
# Hata durumunda anında geri dönüş
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

## 4. Canary Deployment

❌ **Yanlış Kullanım:** Yeni versiyonu doğrudan tüm kullanıcılara açmak.

```bash
kubectl set image deployment/myapp myapp=myapp:v2
# Tüm kullanıcılar yeni versiyonu görür, hata varsa herkesi etkiler
```

✅ **İdeal Kullanım:** Canary deployment ile trafiği kademeli artırın.

```yaml
# %10 trafik yeni versiyona
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # Ana deployment 9 replica
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
        - name: myapp
          image: myapp:v2
```

```csharp
// Uygulama tarafında canary kontrolü
app.Use(async (context, next) =>
{
    var isCanary = context.Request.Headers.ContainsKey("X-Canary");
    context.Items["IsCanary"] = isCanary;
    await next();
});
```

---

## 5. Graceful Shutdown Yapmamak

❌ **Yanlış Kullanım:** Uygulamayı anında kapatmak.

```csharp
// Çalışan istekler kesilir, veri kaybı olabilir
// SIGTERM gelince uygulama hemen kapanır
```

✅ **İdeal Kullanım:** Graceful shutdown ile mevcut işlemleri tamamlayın.

```csharp
builder.Services.AddHostedService<GracefulShutdownService>();

public class GracefulShutdownService : IHostedService
{
    private readonly IHostApplicationLifetime _lifetime;
    private readonly ILogger<GracefulShutdownService> _logger;

    public GracefulShutdownService(IHostApplicationLifetime lifetime, ILogger<GracefulShutdownService> logger)
    {
        _lifetime = lifetime;
        _logger = logger;
    }

    public Task StartAsync(CancellationToken ct)
    {
        _lifetime.ApplicationStopping.Register(() =>
        {
            _logger.LogInformation("Uygulama kapatılıyor, mevcut istekler tamamlanıyor...");
            // Yeni istek kabul etmeyi durdur, mevcut isteklerin tamamlanmasını bekle
        });

        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

```yaml
# Kubernetes'te graceful shutdown süresi
spec:
  terminationGracePeriodSeconds: 30
```