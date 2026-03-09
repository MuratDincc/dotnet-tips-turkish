# CI/CD Pipeline

CI/CD pipeline, kod değişikliklerinin otomatik olarak test edilip deploy edilmesini sağlar; yanlış yapılandırma güvenilmez build'lere ve riskli deployment'lara yol açar.

---

## 1. Test Olmadan Deploy Etmek

❌ **Yanlış Kullanım:** Testleri atlayarak doğrudan deploy yapmak.

```yaml
# GitHub Actions
name: Deploy
on: push
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet publish -c Release
      - run: ./deploy.sh
      # Testler çalıştırılmıyor, hatalı kod production'a gidebilir
```

✅ **İdeal Kullanım:** Build, test ve deploy aşamalarını sıralı olarak çalıştırın.

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - run: dotnet restore
      - run: dotnet build --no-restore
      - run: dotnet test --no-build --verbosity normal

  deploy:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet publish -c Release -o ./publish
      - run: ./deploy.sh
```

---

## 2. Build Artifact'larını Cache'lememek

❌ **Yanlış Kullanım:** Her build'de NuGet paketlerini yeniden indirmek.

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - run: dotnet restore  # Her seferinde tüm paketler indirilir
      - run: dotnet build
      - run: dotnet test
```

✅ **İdeal Kullanım:** NuGet cache'ini kullanarak build süresini kısaltın.

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: nuget-

      - run: dotnet restore
      - run: dotnet build --no-restore
      - run: dotnet test --no-build
```

---

## 3. Secrets'ı Pipeline'da Açık Bırakmak

❌ **Yanlış Kullanım:** Secrets'ı environment variable veya komut satırında açık yazmak.

```yaml
jobs:
  deploy:
    steps:
      - run: dotnet ef database update --connection "Server=prod;Password=MySecret123"
      - run: echo ${{ secrets.DB_PASSWORD }}  # Log'a yazdırılır!
```

✅ **İdeal Kullanım:** Secrets'ı güvenli şekilde yönetin.

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - uses: actions/checkout@v4

      - run: dotnet ef database update
        env:
          ConnectionStrings__Default: ${{ secrets.DB_CONNECTION_STRING }}

      - name: Deploy to Azure
        uses: azure/webapps-deploy@v3
        with:
          app-name: my-app
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: ./publish
```

---

## 4. Branch Stratejisi Olmadan CI/CD

❌ **Yanlış Kullanım:** Her push'ta production'a deploy yapmak.

```yaml
on:
  push:
    branches: ["*"]  # Her branch'ten deploy olabilir

jobs:
  deploy:
    steps:
      - run: ./deploy-to-production.sh
```

✅ **İdeal Kullanım:** Branch stratejisi ile ortam bazlı deploy yapın.

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: dotnet test

  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/develop'
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

---

## 5. Rollback Stratejisi Olmamak

❌ **Yanlış Kullanım:** Hatalı deploy sonrası geri alma planı olmamak.

```yaml
jobs:
  deploy:
    steps:
      - run: dotnet publish -c Release -o ./publish
      - run: ./deploy.sh
      # Deploy başarısız olursa ne yapılacağı belirsiz
```

✅ **İdeal Kullanım:** Versiyonlama ve rollback mekanizması kurun.

```yaml
jobs:
  deploy:
    steps:
      - uses: actions/checkout@v4

      - name: Build and tag
        run: |
          VERSION=${{ github.sha }}
          docker build -t myapp:$VERSION .
          docker tag myapp:$VERSION myregistry/myapp:$VERSION
          docker push myregistry/myapp:$VERSION

      - name: Deploy with rollback
        run: |
          PREVIOUS=$(kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "Previous version: $PREVIOUS" >> $GITHUB_STEP_SUMMARY
          kubectl set image deployment/myapp myapp=myregistry/myapp:${{ github.sha }}
          kubectl rollout status deployment/myapp --timeout=120s || kubectl rollout undo deployment/myapp
```