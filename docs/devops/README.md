# DevOps — Docker / Kubernetes

## Overzicht

Docker containeriseert applicaties zodat ze overal identiek draaien. Kubernetes (K8s) orchestreert die containers op schaal. Samen vormen ze de ruggengraat van moderne cloud-native deployments.

---

## Docker

### Dockerfile voor ASP.NET Core

```dockerfile
# Multi-stage build — klein productie-image
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY ["MyApp.API/MyApp.API.csproj", "MyApp.API/"]
COPY ["MyApp.Application/MyApp.Application.csproj", "MyApp.Application/"]
COPY ["MyApp.Infrastructure/MyApp.Infrastructure.csproj", "MyApp.Infrastructure/"]
RUN dotnet restore "MyApp.API/MyApp.API.csproj"

COPY . .
RUN dotnet publish "MyApp.API/MyApp.API.csproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
EXPOSE 8080

# Non-root user voor security
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.API.dll"]
```

### Dockerfile voor Angular

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine AS runtime
COPY --from=build /app/dist/my-app/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### nginx.conf voor Angular SPA

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # SPA routing
    }

    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    gzip on;
    gzip_types text/plain application/javascript text/css application/json;
}
```

---

## Docker Compose (lokale ontwikkeling)

```yaml
# docker-compose.yml
services:
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "Dev@Passw0rd"
      MSSQL_PID: Developer
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    healthcheck:
      test: ["CMD", "/opt/mssql-tools/bin/sqlcmd", "-S", "localhost", "-U", "sa", "-P", "Dev@Passw0rd", "-Q", "SELECT 1"]
      interval: 10s
      retries: 5

  backend:
    build:
      context: .
      dockerfile: src/MyApp.API/Dockerfile
    environment:
      ConnectionStrings__DefaultConnection: "Server=db;Database=MyAppDb;User Id=sa;Password=Dev@Passw0rd;TrustServerCertificate=true"
      ASPNETCORE_ENVIRONMENT: Development
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "4200:80"
    depends_on:
      - backend

volumes:
  sqldata:
```

---

## Kubernetes

### Deployment

```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: myregistry.azurecr.io/myapp-backend:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: connection-string
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Service

```yaml
# k8s/backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### Ingress (NGINX)

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
```

### Secrets & ConfigMaps

```yaml
# Nooit plaintext secrets in Git! Gebruik Azure Key Vault of sealed-secrets
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: myapp
type: Opaque
stringData:
  connection-string: "Server=myserver;Database=mydb;..."

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  FeatureFlags__NewUI: "true"
```

---

## CI/CD Pipeline (Azure DevOps)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main, develop]

variables:
  imageTag: $(Build.BuildId)
  acrName: myregistry

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: DotNetCoreCLI@2
            displayName: Test
            inputs:
              command: test
              projects: '**/*.Tests.csproj'

          - task: Docker@2
            displayName: Build & Push
            inputs:
              containerRegistry: myACRServiceConnection
              repository: myapp-backend
              command: buildAndPush
              tags: $(imageTag)

  - stage: Deploy
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployToK8s
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  inputs:
                    action: deploy
                    manifests: k8s/*.yaml
                    containers: myregistry.azurecr.io/myapp-backend:$(imageTag)
```

---

## Essentiële kubectl commando's

```bash
# Bekijk resources
kubectl get pods -n myapp
kubectl get deployments -n myapp
kubectl describe pod <pod-name> -n myapp

# Logs bekijken
kubectl logs <pod-name> -n myapp --tail=100 -f

# In een pod stappen
kubectl exec -it <pod-name> -n myapp -- /bin/sh

# Rolling update
kubectl set image deployment/backend backend=myregistry.azurecr.io/myapp-backend:1.0.1 -n myapp

# Rollback
kubectl rollout undo deployment/backend -n myapp

# Schalen
kubectl scale deployment/backend --replicas=5 -n myapp

# Port forward voor lokaal debuggen
kubectl port-forward pod/<pod-name> 8080:8080 -n myapp
```

---

## Best Practices

- Gebruik altijd **multi-stage Dockerfiles** — kleine images, snellere deploys
- Draai containers als **non-root user**
- Definieer altijd **resource requests & limits** in K8s
- Gebruik **liveness en readiness probes** — K8s weet dan wanneer een pod echt klaar is
- Secrets nooit in Git — gebruik **Azure Key Vault** of **Sealed Secrets**
- **Namespaces** per omgeving (dev, staging, production)
- Tag images met buildnummer, nooit `:latest` in productie

---

*[← Terug naar overzicht](../../README.md)*
