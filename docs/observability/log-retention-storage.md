# Log Retention ve Storage Stratejileri

Log saklama ve depolama stratejileri, maliyeti kontrol altında tutarken gerekli verilere erişimi sağlar; yanlış yönetim disk tükenmesine veya veri kaybına yol açar.

---

## 1. Retention Policy Belirlememek

❌ **Yanlış Kullanım:** Tüm logları süresiz saklamak.

```csharp
builder.Host.UseSerilog((context, config) => config
    .WriteTo.File("logs/app.log"));
// Loglar sonsuza kadar birikir, disk dolar
```

✅ **İdeal Kullanım:** Seviye bazlı retention policy belirleyin.

```csharp
builder.Host.UseSerilog((context, config) => config
    .WriteTo.File("logs/app-.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,     // 30 gün
        fileSizeLimitBytes: 100_000_000, // 100MB per dosya
        rollOnFileSizeLimit: true)
    .WriteTo.File("logs/error-.log",
        restrictedToMinimumLevel: LogEventLevel.Error,
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 90));   // Hatalar 90 gün

// Elasticsearch ILM
// Hot:   7 gün  - SSD, hızlı arama
// Warm:  30 gün - HDD, yavaş ama erişilebilir
// Cold:  90 gün - compress, nadiren erişim
// Delete: 90 gün sonra sil
```

---

## 2. Log Volume Kontrolü Yapmamak

❌ **Yanlış Kullanım:** Her isteği detaylı loglamak.

```csharp
app.Use(async (context, next) =>
{
    _logger.LogInformation("Request: {Method} {Path} Headers: {Headers} Body: {Body}",
        context.Request.Method, context.Request.Path,
        context.Request.Headers, await ReadBodyAsync(context));
    await next();
    _logger.LogInformation("Response: {StatusCode} Body: {Body}",
        context.Response.StatusCode, responseBody);
    // Günde milyonlarca log satırı, storage maliyeti patlar
});
```

✅ **İdeal Kullanım:** Akıllı filtreleme ile log hacmini kontrol edin.

```csharp
builder.Host.UseSerilog((context, config) => config
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning)
    .MinimumLevel.Override("System.Net.Http", LogEventLevel.Warning)
    .Filter.ByExcluding(logEvent =>
    {
        if (logEvent.Properties.TryGetValue("RequestPath", out var path))
        {
            var pathStr = path.ToString();
            return pathStr.Contains("/health") || pathStr.Contains("/metrics");
        }
        return false;
    }));
```

---

## 3. Tek Depolama Katmanı Kullanmak

❌ **Yanlış Kullanım:** Tüm logları aynı storage'da tutmak.

```yaml
elasticsearch:
  volumes:
    - es-data:/usr/share/elasticsearch/data
  # Tüm loglar aynı SSD'de, maliyet yüksek
```

✅ **İdeal Kullanım:** Tiered storage ile maliyeti optimize edin.

```yaml
# Elasticsearch ILM Policy
# PUT _ilm/policy/log-tiered-storage
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": { "max_size": "10gb", "max_age": "1d" },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## 4. Backup Stratejisi Olmamak

❌ **Yanlış Kullanım:** Log verilerini yedeklememek.

```
# Elasticsearch node çöktüğünde tüm loglar kaybolur
# Replica olmadan çalışma
```

✅ **İdeal Kullanım:** Snapshot ve backup stratejisi uygulayın.

```json
// PUT _snapshot/log-backup
{
  "type": "fs",
  "settings": {
    "location": "/backup/elasticsearch",
    "compress": true
  }
}

// PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 2 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "log-backup",
  "config": {
    "indices": ["app-*"],
    "ignore_unavailable": true
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 30
  }
}
```

---

## 5. Log Archival Yapmamak

❌ **Yanlış Kullanım:** Eski logları silmek veya erişilemez bırakmak.

```csharp
// 30 günden eski loglar siliniyor
// Compliance gereksinimleri karşılanamıyor
// Geçmiş sorunlar analiz edilemiyor
```

✅ **İdeal Kullanım:** Eski logları ucuz storage'a arşivleyin.

```csharp
public class LogArchivalService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var timer = new PeriodicTimer(TimeSpan.FromDays(1));

        while (await timer.WaitForNextTickAsync(ct))
        {
            var oldIndices = await GetIndicesOlderThan(TimeSpan.FromDays(60));

            foreach (var index in oldIndices)
            {
                // S3/Blob Storage'a arşivle
                await _archiveService.ArchiveIndexAsync(index);
                // Elasticsearch'ten sil
                await _elasticClient.Indices.DeleteAsync(index);

                _logger.LogInformation("Index arşivlendi ve silindi: {Index}", index);
            }
        }
    }
}
```
