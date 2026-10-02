# 🚀 DevOps & Infrastructure

This document covers the complete DevOps setup for the Webhook Delivery Platform — including Docker, Kubernetes, ArgoCD, and GitHub Actions CI/CD pipelines.

---

## 📁 Directory Structure



---

## 🐳 Docker

### Prerequisites
- Docker Desktop installed and running
- Docker Compose v2+

### Environment Variables

Create a `.env` file at the project root:

```env
# JWT
JWT_SECRET=supersecretkey
INTERNAL_SECRET=supersecretinternal

# PostgreSQL - Neon
AUTH_POSTGRES_URI=postgresql://neondb_owner:password@ep-raspy-frog.neon.tech/auth_db?sslmode=require&channel_binding=require
WEBHOOK_POSTGRES_URI=postgresql://neondb_owner:password@ep-raspy-frog.neon.tech/webhook_db?sslmode=require&channel_binding=require
DELIVERY_POSTGRES_URI=postgresql://neondb_owner:password@ep-raspy-frog.neon.tech/delivery_db?sslmode=require&channel_binding=require

# MongoDB - Atlas
EVENT_MONGO_URI=mongodb+srv://user:password@cluster0.mongodb.net/event-db
DLQ_MONGO_URI=mongodb+srv://user:password@cluster0.mongodb.net/dlq-db
LOGS_MONGO_URI=mongodb+srv://user:password@cluster0.mongodb.net/logs-db
NOTIFICATION_MONGO_URI=mongodb+srv://user:password@cluster0.mongodb.net/notification-db

# Email
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USER=your-email@gmail.com
EMAIL_PASS=your-app-password
```

### Run with Docker Compose

```bash
# Build and start all services
docker compose up --build

# Run in background
docker compose up --build -d

# Check all containers
docker compose ps

# View logs
docker compose logs -f
docker compose logs -f api-gateway
docker compose logs -f delivery-service

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# Restart a specific service
docker compose restart api-gateway

# Rebuild a specific service after code change
docker compose up --build api-gateway
```

### Startup Order




### Verify Everything is Running

```bash
# Health checks
curl http://localhost:5000/health   # API Gateway
curl http://localhost:5001/health   # Auth Service
curl http://localhost:5002/health   # Webhook Service
curl http://localhost:5003/health   # Event Service
curl http://localhost:5004/health   # Delivery Service
curl http://localhost:5006/health   # DLQ Service
curl http://localhost:5008/health   # Logs Service
curl http://localhost:5009/health   # Rate Limit Service
```

---

## ☸️ Kubernetes

### Prerequisites
- Kubernetes cluster (minikube / kind / cloud provider)
- kubectl configured
- Metrics Server installed for HPA

### Install Metrics Server (for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Apply Everything in Order

```bash
# 1. Base — namespace, configmap, secrets
kubectl apply -f k8s/base.yml

# 2. Infrastructure
kubectl apply -f k8s/redis.yml
kubectl apply -f k8s/kafka.yml

# 3. Wait for Kafka to be ready
kubectl wait --for=condition=available deployment/kafka \
  -n webhook-platform --timeout=120s

# 4. All microservices
kubectl apply -f k8s/auth-service.yml
kubectl apply -f k8s/webhook-service.yml
kubectl apply -f k8s/event-service.yml
kubectl apply -f k8s/delivery-service.yml
kubectl apply -f k8s/retry-service.yml
kubectl apply -f k8s/dlq-service.yml
kubectl apply -f k8s/notification-service.yml
kubectl apply -f k8s/logs-service.yml
kubectl apply -f k8s/rate-limit-service.yml

# 5. API Gateway last
kubectl apply -f k8s/api-gateway.yml

# 6. Verify everything
kubectl get all -n webhook-platform
kubectl get hpa -n webhook-platform
```

### Kubernetes Architecture

Every service has three Kubernetes resources defined in a single file:

| Resource | Purpose |
|---|---|
| **Deployment** | Manages pods, rolling updates, replicas |
| **Service** | Internal DNS and load balancing between pods |
| **HorizontalPodAutoscaler** | Auto scales pods based on CPU and memory |

### Service Port Reference

| Service | Port | Replicas (min/max) |
|---|---|---|
| API Gateway | 5000 | 2 / 10 |
| Auth Service | 5001 | 2 / 10 |
| Webhook Service | 5002 | 2 / 10 |
| Event Service | 5003 | 2 / 15 |
| Delivery Service | 5004 | 3 / 20 |
| Retry Service | 5005 | 2 / 10 |
| DLQ Service | 5006 | 2 / 8 |
| Notification Service | 5007 | 2 / 8 |
| Logs Service | 5008 | 2 / 8 |
| Rate Limit Service | 5009 | 2 / 10 |

### Rolling Update Strategy

Every deployment uses RollingUpdate with:
- `maxSurge: 1` — one extra pod spun up during update
- `maxUnavailable: 0` — zero downtime, no pods removed until new ones are ready

### HPA Scaling Rules

| Service | Scale Up Trigger | Scale Down Trigger |
|---|---|---|
| API Gateway | CPU > 60% | CPU < 60% for 5 min |
| Delivery Service | CPU > 60% | CPU < 60% for 5 min |
| Event Service | CPU > 70% | CPU < 70% for 5 min |
| All others | CPU > 70% | CPU < 70% for 5 min |

Scale up adds 2-3 pods every 60 seconds. Scale down removes 1 pod every 120 seconds with a 5 minute stabilization window to prevent thrashing.

### Useful kubectl Commands

```bash
# Watch all pods
kubectl get pods -n webhook-platform -w

# Check HPA status
kubectl get hpa -n webhook-platform

# View logs
kubectl logs -f deployment/auth-service -n webhook-platform
kubectl logs -f deployment/delivery-service -n webhook-platform

# Describe a pod for debugging
kubectl describe pod <pod-name> -n webhook-platform

# Rolling restart after new image push
kubectl rollout restart deployment/auth-service -n webhook-platform
kubectl rollout restart deployment/api-gateway -n webhook-platform

# Check rollout status
kubectl rollout status deployment/auth-service -n webhook-platform

# Rollback if something goes wrong
kubectl rollout undo deployment/auth-service -n webhook-platform

# Scale manually
kubectl scale deployment/delivery-service --replicas=5 -n webhook-platform

# Get resource usage
kubectl top pods -n webhook-platform
kubectl top nodes

# Delete everything
kubectl delete namespace webhook-platform
```

### Kafka Topic Setup

A Kubernetes Job automatically creates all required Kafka topics on startup:

| Topic | Partitions | Purpose |
|---|---|---|
| events | 3 | New events from Event Service |
| delivery | 3 | Retry attempts from Retry Service |
| retry | 3 | Failed deliveries from Delivery Service |
| dlq | 1 | Events that exhausted all retries |
| logs | 3 | Aggregated logs from all services |
| user | 1 | User registration data for Notification Service |

---

## 🔄 CI/CD — GitHub Actions

### How It Works

Every service has its own GitHub Actions workflow file. When code is pushed to the `main` branch and files in that service's directory change, the pipeline runs automatically.

### Pipeline Stages



### GitHub Secrets Required

Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Your Docker Hub access token |
| `SERVER_HOST` | Your server IP address |
| `SERVER_USER` | Your server SSH username |
| `SERVER_SSH_KEY` | Your server private SSH key |

### Workflow Files

| Service | Workflow File | Triggers On |
|---|---|---|
| Auth Service | `.github/workflows/auth-service.yml` | `auth-service/**` or `shared/**` |
| Webhook Service | `.github/workflows/webhook-service.yml` | `webhook-service/**` or `shared/**` |
| Event Service | `.github/workflows/event-service.yml` | `event-service/**` or `shared/**` |
| Delivery Service | `.github/workflows/delivery-service.yml` | `delivery-service/**` or `shared/**` |
| Retry Service | `.github/workflows/retry-service.yml` | `retry-service/**` or `shared/**` |
| DLQ Service | `.github/workflows/dlq-service.yml` | `dlq-service/**` or `shared/**` |
| Rate Limit Service | `.github/workflows/rate-limit-service.yml` | `rate-limit-service/**` |

### Path-Based Triggering

Each workflow only runs when files in that specific service change. Pushing to `auth-service` does not trigger the `delivery-service` pipeline. This saves CI minutes and keeps pipelines fast and focused.

---

## 🐙 ArgoCD — GitOps Continuous Deployment

### What is ArgoCD

ArgoCD implements the GitOps pattern. Your GitHub repository is the single source of truth. ArgoCD continuously watches the repository and ensures the Kubernetes cluster matches exactly what is defined in the YAML files. Any drift is automatically corrected.

### Install ArgoCD

```bash
# Create ArgoCD namespace and install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server \
  -n argocd --timeout=300s

# Get initial admin password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Username: admin
# Password: from command above
```

### Setup ArgoCD

```bash
# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password <your-password> \
  --insecure

# Add your GitHub repository
argocd repo add https://github.com/your-username/your-repo.git \
  --username your-github-username \
  --password your-github-token

# Apply namespace and project
kubectl apply -f argocd/namespace.yml
kubectl apply -f argocd/project.yml

# Deploy everything using App of Apps pattern
kubectl apply -f argocd/app-of-apps.yml
```

### App of Apps Pattern

The `app-of-apps.yml` is a single ArgoCD Application that manages all other Applications. When applied, ArgoCD automatically discovers and deploys every service defined in the `argocd/` directory.