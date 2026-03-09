# Dockerfile Best Practices

Dockerfile yapılandırması, container image boyutunu, build süresini ve güvenliği doğrudan etkiler; yanlış yapılandırma şişkin image'lere ve güvenlik açıklarına yol açar.

---

## 1. Multi-Stage Build Kullanmamak

❌ **Yanlış Kullanım:** Tek stage ile SDK'yı production image'ine dahil etmek.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o /app/publish
ENTRYPOINT ["dotnet", "/app/publish/MyApp.dll"]
# Image boyutu: ~700MB+ (SDK dahil)
```

✅ **İdeal Kullanım:** Multi-stage build ile sadece runtime image'ini kullanın.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
# Image boyutu: ~200MB (sadece runtime)
```

---

## 2. Layer Cache'i Bozmak

❌ **Yanlış Kullanım:** Tüm dosyaları restore'dan önce kopyalamak.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish
# Herhangi bir dosya değişikliğinde restore tekrar çalışır
```

✅ **İdeal Kullanım:** Önce csproj dosyalarını kopyalayarak restore cache'ini koruyun.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY *.sln .
COPY src/MyApp.Api/*.csproj src/MyApp.Api/
COPY src/MyApp.Core/*.csproj src/MyApp.Core/
RUN dotnet restore

COPY . .
RUN dotnet publish src/MyApp.Api -c Release -o /app/publish --no-restore
```

---

## 3. Root Kullanıcı ile Çalıştırmak

❌ **Yanlış Kullanım:** Container'ı root kullanıcı ile çalıştırmak.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
# Root olarak çalışır - güvenlik riski
```

✅ **İdeal Kullanım:** Non-root kullanıcı ile çalıştırın.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

RUN adduser --disabled-password --gecos "" appuser
USER appuser

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## 4. .dockerignore Kullanmamak

❌ **Yanlış Kullanım:** Gereksiz dosyaları image'e dahil etmek.

```dockerfile
COPY . .
# bin/, obj/, .git/, node_modules/, test projeleri hepsi kopyalanır
# Build context gereksiz büyür
```

✅ **İdeal Kullanım:** .dockerignore ile gereksiz dosyaları hariç tutun.

```dockerfile
# .dockerignore
**/bin/
**/obj/
**/.git
**/node_modules
**/*.md
**/tests/
**/.vs/
**/.idea/
**/docker-compose*.yml
**/.env
```

---

## 5. Health Check Tanımlamamak

❌ **Yanlış Kullanım:** Container sağlık kontrolü olmadan çalıştırmak.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
# Orchestrator uygulama sağlığını bilmez
```

✅ **İdeal Kullanım:** Dockerfile'da HEALTHCHECK tanımlayın.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

USER appuser
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## 6. Image Tag'i Sabitlememek

❌ **Yanlış Kullanım:** `latest` tag'ini kullanmak.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:latest
# Hangi versiyon gelecek belli değil, build tekrarlanabilir değil
```

✅ **İdeal Kullanım:** Spesifik versiyon tag'i kullanın.

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0.11-bookworm-slim
# Tekrarlanabilir build, bilinen versiyon
```