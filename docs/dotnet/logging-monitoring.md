# Logging ve Monitoring

Logging ve monitoring, uygulamanızın durumu hakkında bilgi toplamak, sorunları tespit etmek ve performans optimizasyonu yapmak için kritik öneme sahiptir. Yanlış uygulamalar log karmaşasına veya yetersiz izlemeye yol açabilir.

---

## 1. Yetersiz Log Seviyeleri

❌ **Yanlış Kullanım:** Tüm log mesajlarını aynı seviyede kaydetmek.

```csharp
logger.LogError("Uygulama başlatılıyor.");
logger.LogError("Bir hata oluştu.");
logger.LogError("Bağlantı sağlandı.");
```

✅ **İdeal Kullanım:** Log seviyelerini olayın önem derecesine göre ayarlayın.

```csharp
logger.LogInformation("Uygulama başlatılıyor.");
logger.LogError("Bir hata oluştu.");
logger.LogDebug("Bağlantı sağlandı.");
```

---

## 2. Hatalı Günlükleme Formatı

❌ **Yanlış Kullanım:** Log mesajlarını okunması zor formatlarda kaydetmek.

```csharp
logger.LogInformation("Error123: ModuleX Failed at StepY.");
```

✅ **İdeal Kullanım:** Yapılandırılmış loglama kullanarak okunabilir ve analiz edilebilir log formatları oluşturun.

```csharp
logger.LogInformation("Module {Module} failed at step {Step}.", "X", "Y");
```

---

## 3. Logların Gereksiz Detay İçermesi

❌ **Yanlış Kullanım:** Her ayrıntıyı loglamak.

```csharp
logger.LogInformation("Kullanıcı ID: 123, İsim: Murat Dinc, IP: 192.168.1.1, Tarayıcı: Chrome...");
```

✅ **İdeal Kullanım:** Önemli bilgileri kaydedin, hassas bilgileri hariç tutun.

```csharp
logger.LogInformation("Kullanıcı ID: {UserId} oturum açtı.", 123);
```

---

## 4. Log Rotation Kullanılmaması

❌ **Yanlış Kullanım:** Log dosyalarını döndürmeden sürekli büyütmek.

```plaintext
app.log // Sonsuza kadar büyüyen log dosyası
```

✅ **İdeal Kullanım:** Log rotation özelliğini etkinleştirin.

- **Örnek:** Serilog ile log dosyalarını döndürmek.

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();
```

---

## 5. Merkezi Log Yönetimi Eksikliği

❌ **Yanlış Kullanım:** Logların yalnızca lokal olarak saklanması.

```plaintext
local-logs/app.log
```

✅ **İdeal Kullanım:** Merkezi bir log yönetim sistemi kullanarak logları bir araya getirin.

- **Azure Application Insights**
- **Elastic Stack (ELK)**
- **Datadog**

```csharp
services.AddApplicationInsightsTelemetry();
```

---

## 6. Monitoring'in Eksikliği

❌ **Yanlış Kullanım:** Uygulamanın performansını ve durumunu izlememek.

✅ **İdeal Kullanım:** Performans ve hata izleme araçlarını entegre edin.

- **Prometheus ve Grafana**
- **Azure Monitor**
- **New Relic**

---

## 7. Hatalı Loglama Frekansı

❌ **Yanlış Kullanım:** Sıkça gerçekleşen olayları aşırı loglamak.

```csharp
for (int i = 0; i < 10000; i++)
{
    logger.LogInformation("İşlem tamamlandı.");
}
```

✅ **İdeal Kullanım:** Önemli olayları loglayın ve loglama frekansını kontrol edin.

```csharp
if (transactionCount % 100 == 0)
{
    logger.LogInformation("{TransactionCount} işlem tamamlandı.", transactionCount);
}
```

---

## 8. Loglarda Hassas Bilgilerin Saklanması

❌ **Yanlış Kullanım:** Şifreler veya hassas bilgileri loglara dahil etmek.

```csharp
logger.LogInformation("Kullanıcı şifresi: {Password}", "123456");
```

✅ **İdeal Kullanım:** Hassas bilgileri hariç tutun.

```csharp
logger.LogInformation("Kullanıcı oturum açtı: {UserId}", userId);
```