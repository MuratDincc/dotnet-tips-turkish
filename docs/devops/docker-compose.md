# Docker Compose

Docker Compose, çoklu container ortamlarını yönetir; yanlış yapılandırma servis bağımlılık sorunlarına ve veri kaybına yol açar.

---

## 1. Servis Bağımlılıklarını Yönetmemek

❌ **Yanlış Kullanım:** `depends_on` kullanmadan servis başlatmak.

```yaml
services:
  api:
    build: .
    ports:
      - "5000:8080"
  db:
    image: postgres:16
  redis:
    image: redis:7
# API, veritabanı hazır olmadan başlar ve crash olur
```

✅ **İdeal Kullanım:** `depends_on` ile health check bazlı bağımlılık tanımlayın.

```yaml
services:
  api:
    build: .
    ports:
      - "5000:8080"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=secret

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

---

## 2. Volume Olmadan Veritabanı Çalıştırmak

❌ **Yanlış Kullanım:** Veritabanı verisini container içinde tutmak.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
# Container silindiğinde tüm veri kaybolur
```

✅ **İdeal Kullanım:** Named volume ile veriyi persist edin.

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres-data:
```

---

## 3. Hardcoded Değerler Kullanmak

❌ **Yanlış Kullanım:** Şifreleri ve konfigürasyonu doğrudan yaml'a yazmak.

```yaml
services:
  api:
    environment:
      - DB_PASSWORD=SuperSecret123!
      - JWT_SECRET=MyJwtSecretKey2024
      - SMTP_PASSWORD=email_password
```

✅ **İdeal Kullanım:** .env dosyası ve secrets ile yönetin.

```yaml
services:
  api:
    env_file:
      - .env
    environment:
      - ConnectionStrings__Default=Host=db;Database=${DB_NAME};Username=${DB_USER};Password=${DB_PASSWORD}
    secrets:
      - jwt_secret

secrets:
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

```bash
# .env
DB_NAME=myapp
DB_USER=postgres
DB_PASSWORD=SuperSecret123!
```

---

## 4. Network İzolasyonu Yapmamak

❌ **Yanlış Kullanım:** Tüm servislerin aynı default network'te olması.

```yaml
services:
  api:
    ports:
      - "5000:8080"
  admin-api:
    ports:
      - "5001:8080"
  db:
    ports:
      - "5432:5432"
  redis:
    ports:
      - "6379:6379"
# Tüm servisler birbirine erişebilir, DB dışarıya açık
```

✅ **İdeal Kullanım:** Ağ izolasyonu ile servisleri segmente edin.

```yaml
services:
  api:
    networks:
      - frontend
      - backend
    ports:
      - "5000:8080"

  admin-api:
    networks:
      - admin
      - backend

  db:
    networks:
      - backend
    # Port dışarıya açılmaz, sadece backend network'ü erişir

  redis:
    networks:
      - backend

networks:
  frontend:
  backend:
  admin:
```

---

## 5. Resource Limitleri Belirlememek

❌ **Yanlış Kullanım:** Container'lara sınırsız kaynak vermek.

```yaml
services:
  api:
    build: .
  worker:
    build: ./worker
# Bir container tüm host kaynaklarını tüketebilir
```

✅ **İdeal Kullanım:** CPU ve memory limitleri tanımlayın.

```yaml
services:
  api:
    build: .
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M

  worker:
    build: ./worker
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
        reservations:
          cpus: "0.1"
          memory: 64M
```