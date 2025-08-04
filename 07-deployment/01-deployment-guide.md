# Deployment Guide

This guide covers deploying the BLADE ingestion service to various environments.

## 1. Docker Deployment

### Create Dockerfile

Create `Dockerfile` in the project root:

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git make protoc

# Set working directory
WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Generate proto files
RUN make proto

# Build the application
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o blade-server ./server/main.go

# Final stage
FROM alpine:latest

# Install runtime dependencies
RUN apk --no-cache add ca-certificates tzdata

# Create non-root user
RUN addgroup -g 1000 blade && \
    adduser -D -s /bin/sh -u 1000 -G blade blade

WORKDIR /app

# Copy binary from builder
COPY --from=builder /app/blade-server .
COPY --from=builder /app/swagger ./swagger

# Change ownership
RUN chown -R blade:blade /app

# Switch to non-root user
USER blade

# Expose ports
EXPOSE 9090 9091

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:9091/healthz || exit 1

# Run the application
CMD ["./blade-server"]
```

### Create docker-compose.yml

Create `docker-compose.yml` for full stack deployment:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-blade_user}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-blade_password}
      POSTGRES_DB: ${DB_NAME:-blade_ingestion}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-blade_user} -d ${DB_NAME:-blade_ingestion}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - blade_network

  blade-service:
    build: .
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      # Database configuration
      PGHOST: postgres
      PGPORT: 5432
      PG_DATABASE: ${DB_NAME:-blade_ingestion}
      APP_DB_USER: ${DB_USER:-blade_user}
      APP_DB_ADMIN_PASSWORD: ${DB_PASSWORD:-blade_password}
      DB_SSL_MODE: disable
      
      # Service configuration
      APP_HOST: 0.0.0.0
      GRPC_PORT: 9090
      REST_PORT: 9091
      
      # Databricks configuration
      DATABRICKS_HOST: ${DATABRICKS_HOST}
      DATABRICKS_TOKEN: ${DATABRICKS_TOKEN}
      DATABRICKS_WAREHOUSE_ID: ${DATABRICKS_WAREHOUSE_ID}
      
      # Catalog configuration
      CATALOG_SERVICE_URL: ${CATALOG_SERVICE_URL}
      CATALOG_AUTH_TOKEN: ${CATALOG_AUTH_TOKEN}
      
      # Logging
      LOG_LEVEL: ${LOG_LEVEL:-info}
      LOG_FORMAT: json
      
      # Features
      ENABLE_SWAGGER_UI: ${ENABLE_SWAGGER_UI:-true}
    ports:
      - "9090:9090"  # gRPC
      - "9091:9091"  # REST
    networks:
      - blade_network
    volumes:
      - ./logs:/app/logs

  # Optional: Mock Databricks for testing
  mock-databricks:
    image: mockserver/mockserver:latest
    profiles: ["dev"]
    environment:
      MOCKSERVER_PROPERTY_FILE: /config/mockserver.properties
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/databricks-mock.json
    ports:
      - "8080:1080"
    networks:
      - blade_network
    volumes:
      - ./mock/databricks-mock.json:/config/databricks-mock.json:ro

volumes:
  postgres_data:

networks:
  blade_network:
    driver: bridge
```

### Build and Run with Docker

```bash
# Build the image
docker build -t blade-ingestion-service:latest .

# Run with docker-compose
docker-compose up -d

# View logs
docker-compose logs -f blade-service

# Stop services
docker-compose down
```

## 2. Kubernetes Deployment

### Create Kubernetes Manifests

Create `k8s/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: blade-ingestion
```

Create `k8s/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: blade-config
  namespace: blade-ingestion
data:
  APP_HOST: "0.0.0.0"
  GRPC_PORT: "9090"
  REST_PORT: "9091"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  ENABLE_SWAGGER_UI: "true"
  DB_SSL_MODE: "require"
```

Create `k8s/secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: blade-secrets
  namespace: blade-ingestion
type: Opaque
stringData:
  db-password: "your-db-password"
  databricks-token: "your-databricks-token"
  catalog-token: "your-catalog-token"
```

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blade-ingestion-service
  namespace: blade-ingestion
  labels:
    app: blade-ingestion
spec:
  replicas: 3
  selector:
    matchLabels:
      app: blade-ingestion
  template:
    metadata:
      labels:
        app: blade-ingestion
    spec:
      containers:
      - name: blade-service
        image: blade-ingestion-service:latest
        imagePullPolicy: Always
        ports:
        - name: grpc
          containerPort: 9090
          protocol: TCP
        - name: rest
          containerPort: 9091
          protocol: TCP
        env:
        - name: PGHOST
          value: "postgres-service"
        - name: PGPORT
          value: "5432"
        - name: PG_DATABASE
          value: "blade_ingestion"
        - name: APP_DB_USER
          value: "blade_user"
        - name: APP_DB_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: blade-secrets
              key: db-password
        - name: DATABRICKS_HOST
          value: "https://databricks.example.com"
        - name: DATABRICKS_TOKEN
          valueFrom:
            secretKeyRef:
              name: blade-secrets
              key: databricks-token
        - name: DATABRICKS_WAREHOUSE_ID
          value: "warehouse-123"
        - name: CATALOG_SERVICE_URL
          value: "http://catalog-service:8080/api/v1"
        - name: CATALOG_AUTH_TOKEN
          valueFrom:
            secretKeyRef:
              name: blade-secrets
              key: catalog-token
        envFrom:
        - configMapRef:
            name: blade-config
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: rest
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /healthz
            port: rest
          initialDelaySeconds: 5
          periodSeconds: 5
```

Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blade-ingestion-service
  namespace: blade-ingestion
spec:
  selector:
    app: blade-ingestion
  ports:
  - name: grpc
    port: 9090
    targetPort: grpc
  - name: rest
    port: 9091
    targetPort: rest
  type: ClusterIP
```

Create `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blade-ingestion-ingress
  namespace: blade-ingestion
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - blade-api.example.com
    secretName: blade-tls-cert
  rules:
  - host: blade-api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blade-ingestion-service
            port:
              number: 9091
```

### Deploy to Kubernetes

```bash
# Apply all manifests
kubectl apply -f k8s/

# Check deployment status
kubectl -n blade-ingestion get deployments
kubectl -n blade-ingestion get pods
kubectl -n blade-ingestion get services

# View logs
kubectl -n blade-ingestion logs -f deployment/blade-ingestion-service

# Scale deployment
kubectl -n blade-ingestion scale deployment blade-ingestion-service --replicas=5
```

## 3. Helm Chart

### Create Helm Chart Structure

```bash
helm create blade-ingestion-chart
```

Update `blade-ingestion-chart/values.yaml`:

```yaml
replicaCount: 3

image:
  repository: blade-ingestion-service
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  grpcPort: 9090
  restPort: 9091

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: blade-api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: blade-tls-cert
      hosts:
        - blade-api.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

config:
  logLevel: info
  enableSwagger: true
  
database:
  host: postgres-service
  port: 5432
  name: blade_ingestion
  user: blade_user
  sslMode: require
  
databricks:
  host: https://databricks.example.com
  warehouseId: warehouse-123
  
catalog:
  serviceUrl: http://catalog-service:8080/api/v1

secrets:
  dbPassword: ""
  databricksToken: ""
  catalogToken: ""
```

### Deploy with Helm

```bash
# Install
helm install blade-ingestion ./blade-ingestion-chart \
  --namespace blade-ingestion \
  --create-namespace \
  --set secrets.dbPassword=$DB_PASSWORD \
  --set secrets.databricksToken=$DATABRICKS_TOKEN \
  --set secrets.catalogToken=$CATALOG_TOKEN

# Upgrade
helm upgrade blade-ingestion ./blade-ingestion-chart \
  --namespace blade-ingestion

# Uninstall
helm uninstall blade-ingestion --namespace blade-ingestion
```

## 4. Production Considerations

### Environment Configuration

Create `.env.production`:

```bash
# Database
PGHOST=prod-postgres.example.com
PGPORT=5432
PG_DATABASE=blade_ingestion_prod
APP_DB_USER=blade_prod_user
APP_DB_ADMIN_PASSWORD=${DB_PASSWORD}
DB_SSL_MODE=require

# Service
APP_HOST=0.0.0.0
GRPC_PORT=9090
REST_PORT=9091

# Databricks
DATABRICKS_HOST=https://prod-databricks.example.com
DATABRICKS_TOKEN=${DATABRICKS_TOKEN}
DATABRICKS_WAREHOUSE_ID=prod-warehouse-id

# Catalog
CATALOG_SERVICE_URL=https://catalog.example.com/api/v1
CATALOG_AUTH_TOKEN=${CATALOG_TOKEN}
CATALOG_UPLOAD_TIMEOUT=60s

# Logging
LOG_LEVEL=info
LOG_FORMAT=json

# Features
ENABLE_SWAGGER_UI=false
ENABLE_METRICS=true
ENABLE_TRACING=true
```

### Monitoring and Observability

Add Prometheus metrics endpoint:

```go
// Add to main.go
import "github.com/prometheus/client_golang/prometheus/promhttp"

// In main function
mux.Handle("/metrics", promhttp.Handler())
```

### Security Hardening

1. **Use TLS for all connections**
2. **Implement rate limiting**
3. **Add authentication/authorization**
4. **Scan images for vulnerabilities**
5. **Use network policies in Kubernetes**
6. **Implement secrets rotation**

### Performance Tuning

1. **Database connection pooling**
2. **Redis for caching**
3. **Horizontal scaling**
4. **Query optimization**
5. **Batch processing optimization**

## 5. CI/CD Pipeline

### GitHub Actions Deployment

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      
    steps:
    - uses: actions/checkout@v3
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Kubernetes
      env:
        KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
      run: |
        echo "$KUBE_CONFIG" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig
        
        kubectl set image deployment/blade-ingestion-service \
          blade-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:main \
          -n blade-ingestion
        
        kubectl rollout status deployment/blade-ingestion-service -n blade-ingestion
```

## 6. Troubleshooting

### Common Issues

1. **Database Connection Failed**
   - Check network connectivity
   - Verify credentials
   - Check SSL/TLS settings

2. **Databricks API Errors**
   - Verify token validity
   - Check warehouse status
   - Review API quotas

3. **High Memory Usage**
   - Tune batch sizes
   - Implement pagination
   - Add memory limits

4. **Slow Performance**
   - Add database indexes
   - Optimize queries
   - Scale horizontally

### Debug Commands

```bash
# Check logs
kubectl logs -f deployment/blade-ingestion-service -n blade-ingestion

# Execute into pod
kubectl exec -it deployment/blade-ingestion-service -n blade-ingestion -- /bin/sh

# Port forward for local debugging
kubectl port-forward service/blade-ingestion-service 9090:9090 9091:9091 -n blade-ingestion

# Check resource usage
kubectl top pods -n blade-ingestion
```

## Summary

✅ Docker deployment configuration  
✅ Kubernetes manifests  
✅ Helm chart setup  
✅ Production considerations  
✅ CI/CD pipeline  
✅ Troubleshooting guide  

The BLADE ingestion service is now ready for deployment across various environments!