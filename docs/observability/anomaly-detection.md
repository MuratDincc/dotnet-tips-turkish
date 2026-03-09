# Anomaly Detection ile Otomatik Hata Tespiti

Anomaly detection, normal davranış kalıplarından sapmaları otomatik tespit eder; sabit eşikli uyarılar dinamik sistemlerde yetersiz kalır ve sorunların geç fark edilmesine yol açar.

---

## 1. Sabit Eşik ile Alert Tanımlamak

❌ **Yanlış Kullanım:** Statik threshold ile çalışmak.

```yaml
# alert_rules.yml
- alert: HighLatency
  expr: http_request_duration_seconds > 0.5
  # Gece 03:00'te 0.3s normal, öğlen 12:00'de 0.5s normal
  # Sabit eşik her iki durumda da yanlış sonuç verir
```

✅ **İdeal Kullanım:** Dinamik baseline ile anomali tespiti yapın.

```yaml
# Prometheus alert - son 7 günlük ortalamaya göre
- alert: LatencyAnomaly
  expr: >
    http_request_duration_seconds:p95
    > avg_over_time(http_request_duration_seconds:p95[7d]) * 2
    and
    http_request_duration_seconds:p95
    > avg_over_time(http_request_duration_seconds:p95[7d]) + 2 *
      stddev_over_time(http_request_duration_seconds:p95[7d])
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Latency anomalisi: P95 normal aralığın dışında"
```

---

## 2. Sadece Teknik Metrikleri İzlemek

❌ **Yanlış Kullanım:** İş metriklerinde anomali aramamak.

```csharp
// CPU, Memory, HTTP 500 izleniyor ama:
// - Sipariş sayısı anormal düştüğünde fark edilmiyor
// - Ödeme başarı oranı değiştiğinde alarm yok
// - Kayıt oranı sıfıra düştüğünde kimse bilmiyor
```

✅ **İdeal Kullanım:** İş metriklerinde de anomali tespiti yapın.

```csharp
public class BusinessAnomalyDetector : BackgroundService
{
    private readonly IMetricsService _metrics;
    private readonly IAlertService _alertService;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));

        while (await timer.WaitForNextTickAsync(ct))
        {
            await CheckOrderAnomalyAsync(ct);
            await CheckPaymentAnomalyAsync(ct);
        }
    }

    private async Task CheckOrderAnomalyAsync(CancellationToken ct)
    {
        var currentRate = await _metrics.GetOrderRateAsync(TimeSpan.FromMinutes(15));
        var historicalAvg = await _metrics.GetOrderRateAsync(TimeSpan.FromDays(7));
        var stdDev = await _metrics.GetOrderRateStdDevAsync(TimeSpan.FromDays(7));

        // Z-score hesapla
        var zScore = (currentRate - historicalAvg) / stdDev;

        if (zScore < -2.5) // Normalden çok düşük
        {
            await _alertService.SendAsync(new Alert
            {
                Severity = AlertSeverity.Warning,
                Title = "Sipariş oranı anomalisi",
                Message = $"Mevcut: {currentRate:F1}/dk, Beklenen: {historicalAvg:F1}/dk (z={zScore:F2})"
            });
        }
    }
}
```

---

## 3. Error Rate Spike Tespiti Yapmamak

❌ **Yanlış Kullanım:** Toplam hata sayısına bakmak.

```yaml
- alert: TooManyErrors
  expr: http_server_errors_total > 100
  # Yoğun saatte 100 hata normal olabilir, gece 10 hata bile sorun olabilir
```

✅ **İdeal Kullanım:** Hata oranı değişimini (spike) tespit edin.

```csharp
public class ErrorRateSpikeDetector
{
    public bool IsSpike(double currentErrorRate, double[] recentRates)
    {
        if (recentRates.Length < 10) return false;

        var mean = recentRates.Average();
        var stdDev = Math.Sqrt(recentRates.Average(r => Math.Pow(r - mean, 2)));

        // Modified Z-Score (outlier'lara dayanıklı)
        var median = GetMedian(recentRates);
        var mad = GetMedian(recentRates.Select(r => Math.Abs(r - median)).ToArray());
        var modifiedZScore = 0.6745 * (currentErrorRate - median) / mad;

        return modifiedZScore > 3.5;
    }

    private static double GetMedian(double[] values)
    {
        var sorted = values.OrderBy(v => v).ToArray();
        var mid = sorted.Length / 2;
        return sorted.Length % 2 == 0
            ? (sorted[mid - 1] + sorted[mid]) / 2.0
            : sorted[mid];
    }
}
```

---

## 4. Seasonality'yi Hesaba Katmamak

❌ **Yanlış Kullanım:** Gün içi ve haftalık örüntüleri görmezden gelmek.

```csharp
// Pazar günü trafik %60 düşer ama alarm tetiklenir
// Gece 03:00 trafiği düşük ama "trafik azaldı" uyarısı gelir
```

✅ **İdeal Kullanım:** Zaman bazlı karşılaştırma yapın.

```csharp
public class SeasonalAnomalyDetector
{
    public async Task<AnomalyResult> DetectAsync(string metricName, double currentValue)
    {
        // Aynı saatte, aynı günde geçmiş hafta verileri
        var historicalValues = await _metricsStore.GetHistoricalAsync(
            metricName,
            dayOfWeek: DateTime.UtcNow.DayOfWeek,
            hourOfDay: DateTime.UtcNow.Hour,
            weeksBack: 4);

        var mean = historicalValues.Average();
        var stdDev = CalculateStdDev(historicalValues);
        var zScore = stdDev > 0 ? (currentValue - mean) / stdDev : 0;

        return new AnomalyResult
        {
            IsAnomaly = Math.Abs(zScore) > 2.5,
            ZScore = zScore,
            ExpectedValue = mean,
            ActualValue = currentValue,
            Direction = zScore > 0 ? "above" : "below"
        };
    }
}

// Kullanım
var result = await _detector.DetectAsync("orders_per_minute", currentOrderRate);
if (result.IsAnomaly)
    _logger.LogWarning("Anomali: {Metric} beklenen {Expected:F1}, gerçek {Actual:F1} (z={ZScore:F2})",
        "orders_per_minute", result.ExpectedValue, result.ActualValue, result.ZScore);
```

---

## 5. Anomali Sonrası Otomatik Aksiyon Almamak

❌ **Yanlış Kullanım:** Anomali tespitinden sonra sadece alert göndermek.

```csharp
if (isAnomaly)
    await _slack.SendAsync("Anomali tespit edildi!");
// İnsan müdahalesi beklenecek, gece yarısı ise geç kalınabilir
```

✅ **İdeal Kullanım:** Anomali tipine göre otomatik aksiyon alın.

```csharp
public class AnomalyResponseService
{
    public async Task HandleAnomalyAsync(AnomalyResult anomaly)
    {
        switch (anomaly.Type)
        {
            case AnomalyType.HighErrorRate:
                // Circuit breaker'ı aç
                await _circuitBreaker.OpenAsync(anomaly.AffectedService);
                // Fallback'e yönlendir
                await _featureFlags.EnableAsync("UseFallbackService");
                await _alertService.SendAsync(AlertSeverity.Critical, anomaly);
                break;

            case AnomalyType.HighLatency:
                // Cache TTL'i uzat (DB yükünü azalt)
                await _cacheConfig.ExtendTtlAsync(TimeSpan.FromMinutes(30));
                // Non-critical background job'ları duraklat
                await _jobScheduler.PauseNonCriticalAsync();
                await _alertService.SendAsync(AlertSeverity.Warning, anomaly);
                break;

            case AnomalyType.TrafficSpike:
                // Rate limiter'ı sıkılaştır
                await _rateLimiter.ReduceLimitAsync(0.5);
                // Auto-scale tetikle
                await _scaler.ScaleUpAsync(anomaly.AffectedService, 2);
                await _alertService.SendAsync(AlertSeverity.Warning, anomaly);
                break;
        }
    }
}
```
