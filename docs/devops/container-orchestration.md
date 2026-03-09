# Container Orchestration

Container orchestration, container'ların ölçeklenmesi, dağıtımı ve yönetimini otomatikleştirir; yanlış yapılandırma kaynak israfına ve servis kesintilerine yol açar.

---

## 1. Resource Limitleri Belirlememek

❌ **Yanlış Kullanım:** Container'lara sınırsız kaynak tanımlamak.

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:v1
      # Resource limitleri yok
      # Bir pod tüm node kaynaklarını tüketebilir
```

✅ **İdeal Kullanım:** Request ve limit değerlerini tanımlayın.

```yaml
spec:
  containers:
    - name: myapp
      image: myapp:v1
      resources:
        requests:
          cpu: "250m"
          memory: "256Mi"
        limits:
          cpu: "500m"
          memory: "512Mi"
```

---

## 2. Horizontal Pod Autoscaler Kullanmamak

❌ **Yanlış Kullanım:** Sabit replica sayısı ile çalışmak.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3  # Yük artsa da azalsa da 3 pod
```

✅ **İdeal Kullanım:** HPA ile otomatik ölçekleme yapın.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 3. ConfigMap ve Secret Kullanmamak

❌ **Yanlış Kullanım:** Konfigürasyonu container image'ine gömmek.

```dockerfile
COPY appsettings.Production.json /app/appsettings.json
# Konfigürasyon değişikliği için yeniden image build gerekir
```

✅ **İdeal Kullanım:** ConfigMap ve Secret ile konfigürasyonu dışarıdan yönetin.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  appsettings.json: |
    {
      "Logging": { "LogLevel": { "Default": "Information" } },
      "AllowedHosts": "*"
    }

---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  ConnectionStrings__Default: "Server=db;Database=myapp;Password=secret"

---
spec:
  containers:
    - name: myapp
      envFrom:
        - secretRef:
            name: myapp-secrets
      volumeMounts:
        - name: config
          mountPath: /app/appsettings.Production.json
          subPath: appsettings.json
  volumes:
    - name: config
      configMap:
        name: myapp-config
```

---

## 4. Pod Disruption Budget Tanımlamamak

❌ **Yanlış Kullanım:** Node bakımında tüm pod'ların aynı anda evict edilmesi.

```yaml
# PDB yok
# kubectl drain node-1 çalıştırıldığında tüm pod'lar aynı anda silinebilir
```

✅ **İdeal Kullanım:** PDB ile minimum availability garantisi verin.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

## 5. Namespace İzolasyonu Yapmamak

❌ **Yanlış Kullanım:** Tüm servisleri default namespace'te çalıştırmak.

```yaml
# Tüm servisler default namespace'te
# İzolasyon yok, resource quota yok, güvenlik kontrolü zor
kubectl get pods  # Yüzlerce pod karışık
```

✅ **İdeal Kullanım:** Ortam ve takım bazlı namespace izolasyonu yapın.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```