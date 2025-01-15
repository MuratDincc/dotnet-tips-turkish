# Temporal Tables

Temporal tables, veritabanı tablolarındaki değişikliklerin geçmişini tutarak veri versiyonlamasını mümkün kılar. Bu özellik, hata ayıklama, veri analitiği ve yasal gereksinimler için oldukça faydalıdır. Ancak, yanlış kullanımları depolama ve performans sorunlarına neden olabilir.

---

## 1. Temporal Tables'ı Gereksiz Kullanmak

❌ **Yanlış Kullanım:** Geçmiş verileri gerektirmeyen tablolar için temporal tables kullanmak.

```sql
CREATE TABLE Products
(
    ProductId INT PRIMARY KEY,
    Name NVARCHAR(100),
    Price DECIMAL(10, 2),
    SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START,
    SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
) WITH (SYSTEM_VERSIONING = ON);
```

✅ **İdeal Kullanım:** Sadece geçmiş verilerin kritik olduğu tablolar için temporal tables etkinleştirin.

```sql
CREATE TABLE Orders
(
    OrderId INT PRIMARY KEY,
    CustomerId INT,
    OrderDate DATETIME2,
    SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START,
    SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
) WITH (SYSTEM_VERSIONING = ON);
```

---

## 2. Temporal Table Veri Yönetimini Göz Ardı Etmek

❌ **Yanlış Kullanım:** Temporal tables verilerini otomatik olarak temizlememek.

```sql
-- Tüm geçmiş veriler sonsuza kadar saklanır
SELECT * FROM Orders FOR SYSTEM_TIME ALL;
```

✅ **İdeal Kullanım:** Veritabanı boyutunu yönetmek için geçmiş verileri temizleyin.

```sql
ALTER DATABASE [YourDatabase] SET TEMPORAL_HISTORY_RETENTION_PERIOD = 6 MONTHS;
```

---

## 3. Yanlış Sorgu Yazımı

❌ **Yanlış Kullanım:** Temporal tables için standart sorguları kullanmak.

```sql
SELECT * FROM Orders WHERE OrderDate = '2023-01-01';
```

✅ **İdeal Kullanım:** Temporal tables sorgularında geçmiş verileri açıkça belirtin.

```sql
SELECT * FROM Orders
FOR SYSTEM_TIME AS OF '2023-01-01T00:00:00';
```

---

## 4. Temporal Tables ile Yanlış İlişkiler Kurmak

❌ **Yanlış Kullanım:** Temporal table içeren tabloları yanlış ilişkilerle bağlamak.

```sql
CREATE TABLE OrderDetails
(
    OrderDetailId INT PRIMARY KEY,
    OrderId INT,
    ProductId INT
);
```

✅ **İdeal Kullanım:** İlişkileri temporal table'lara uygun şekilde tasarlayın.

```sql
CREATE TABLE OrderDetails
(
    OrderDetailId INT PRIMARY KEY,
    OrderId INT,
    ProductId INT,
    SysStartTime DATETIME2 GENERATED ALWAYS AS ROW START,
    SysEndTime DATETIME2 GENERATED ALWAYS AS ROW END,
    PERIOD FOR SYSTEM_TIME (SysStartTime, SysEndTime)
) WITH (SYSTEM_VERSIONING = ON);
```

---

## 5. Hata Ayıklama ve Analiz İçin Temporal Tables'ı Kullanmamak

❌ **Yanlış Kullanım:** Temporal tables'ı hata ayıklama veya veri analitiği için kullanmamak.

```sql
-- Geçmiş veriler sorgulanmıyor
SELECT * FROM Orders WHERE OrderId = 101;
```

✅ **İdeal Kullanım:** Temporal tables'ı geçmiş verileri analiz etmek için etkin kullanın.

```sql
SELECT * FROM Orders
FOR SYSTEM_TIME BETWEEN '2023-01-01T00:00:00' AND '2023-01-31T23:59:59';
```

---

## 6. Performans ve Depolama Maliyetlerini Göz Ardı Etmek

❌ **Yanlış Kullanım:** Temporal tables veritabanı boyutunu yönetmemek.

```sql
-- Tüm geçmiş veriler sürekli saklanır
SELECT * FROM Orders FOR SYSTEM_TIME ALL;
```

✅ **İdeal Kullanım:** Performansı optimize etmek için indeksler ve arşivleme kullanın.

```sql
CREATE INDEX IX_Orders_SysStartTime ON Orders (SysStartTime);
```