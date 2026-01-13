# Kubernetes Deployment Guide

This directory contains Kubernetes manifests for deploying the Product App (React + Spring Boot + MySQL) to a Kubernetes cluster.

## 📋 Prerequisites

- **Kubernetes cluster** (Minikube, Kind, or cloud provider)
- **kubectl** CLI tool installed and configured
- **Docker** for building images

### Install Minikube (Windows)

```powershell
choco install minikube
minikube start
```

## 🏗️ Architecture

```
┌─────────────────┐
│   react-app     │  NodePort :30000
│  (Nginx + React)│  
└────────┬────────┘
         │ Proxy /api/*
┌────────▼────────┐
│ spring-boot-app │  ClusterIP :8080
│   (REST API)    │
└────────┬────────┘
         │ JDBC
┌────────▼────────┐
│     mysql       │  ClusterIP :3306
│   (Database)    │
└─────────────────┘
```

## 🚀 Quick Start

### Step 1: Build Docker Images

For **Minikube**, use Minikube's Docker daemon to avoid pushing to a registry:

```powershell
# Point Docker CLI to Minikube's Docker daemon
minikube docker-env | Invoke-Expression

# Build backend image
cd backend
docker build -t product-backend:v1 .

# Build frontend image
cd ../frontend
docker build -t product-frontend:v1 --build-arg REACT_APP_API_BASE_URL="" .

cd ..
```

### Step 2: Deploy to Kubernetes

Apply manifests in the correct order:

```powershell
# Create namespace
kubectl apply -f k8s/namespace.yaml

# Deploy MySQL (database layer)
kubectl apply -f k8s/mysql/

# Deploy Spring Boot (backend layer)
kubectl apply -f k8s/spring-boot/

# Deploy React (frontend layer)
kubectl apply -f k8s/react/
```

### Step 3: Verify Deployment

```powershell
# Check all resources
kubectl get all -n product-app

# Check pod status (wait until all are Running)
kubectl get pods -n product-app

# Expected output:
# NAME                               READY   STATUS    RESTARTS   AGE
# mysql-xxxxxxxxxx-xxxxx             1/1     Running   0          2m
# spring-boot-app-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
# spring-boot-app-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
# react-app-xxxxxxxxxx-xxxxx         1/1     Running   0          30s
# react-app-xxxxxxxxxx-xxxxx         1/1     Running   0          30s
```

### Step 4: Access the Application

**Option 1: Using Minikube service command (Recommended)**

```powershell
minikube service react-service -n product-app
```

This will automatically open your browser to the correct URL.

**Option 2: Using NodePort directly**

```powershell
# Get Minikube IP
minikube ip

# Access at: http://<minikube-ip>:30000
```

**Option 3: Port forwarding**

```powershell
kubectl port-forward -n product-app service/react-service 3000:80
# Access at: http://localhost:3000
```

## 📁 Directory Structure

```
k8s/
├── namespace.yaml              # Namespace definition
├── mysql/
│   ├── secret.yaml            # Database credentials
│   ├── pvc.yaml               # Persistent volume claim
│   ├── deployment.yaml        # MySQL deployment
│   └── service.yaml           # MySQL service
├── spring-boot/
│   ├── deployment.yaml        # Spring Boot deployment
│   └── service.yaml           # Spring Boot service
└── react/
    ├── deployment.yaml        # React + Nginx deployment
    └── service.yaml           # React service (NodePort)
```

## 🔍 Troubleshooting

### Check Pod Logs

```powershell
# List all pods
kubectl get pods -n product-app

# View logs for a specific pod
kubectl logs -n product-app <pod-name>

# Follow logs in real-time
kubectl logs -n product-app <pod-name> -f

# View logs for Spring Boot
kubectl logs -n product-app -l component=spring-boot

# View logs for React
kubectl logs -n product-app -l component=react
```

### Common Issues

**Pods stuck in `Pending` state:**
```powershell
kubectl describe pod -n product-app <pod-name>
# Check Events section for issues (usually resource constraints)
```

**Spring Boot can't connect to MySQL:**
```powershell
# Check if MySQL is ready
kubectl get pods -n product-app -l component=mysql

# Verify MySQL service exists
kubectl get svc -n product-app mysql

# Check Spring Boot logs
kubectl logs -n product-app -l component=spring-boot
```

**Images not found (ImagePullBackOff):**
- Make sure you built images in Minikube's Docker daemon
- Run: `minikube docker-env | Invoke-Expression` before building
- Verify images exist: `docker images | Select-String product`

### Restart Deployments

```powershell
# Restart Spring Boot
kubectl rollout restart deployment/spring-boot-app -n product-app

# Restart React
kubectl rollout restart deployment/react-app -n product-app
```

## 🧪 Testing

### Test Database Connectivity

```powershell
# Connect to MySQL pod
kubectl exec -it -n product-app deployment/mysql -- mysql -u product_user -pproduct_user_password product

# Run SQL query
mysql> SELECT * FROM products;
mysql> exit
```

### Test API Directly

```powershell
# Port forward Spring Boot service
kubectl port-forward -n product-app service/spring-boot-app 8080:8080

# Test API (in another terminal)
curl http://localhost:8080/api/products
```

### Test Full Stack

1. Access the React app via browser
2. Create a new product
3. Verify it appears in the list
4. Delete a Spring Boot pod: `kubectl delete pod -n product-app -l component=spring-boot`
5. Wait for new pod to start
6. Verify data still exists (persistence working)

## 🔧 Configuration

### Resource Limits

Current resource allocations:

| Component    | CPU Request | CPU Limit | Memory Request | Memory Limit |
|--------------|-------------|-----------|----------------|--------------|
| MySQL        | 250m        | 500m      | 256Mi          | 512Mi        |
| Spring Boot  | 500m        | 1000m     | 512Mi          | 1Gi          |
| React        | 100m        | 200m      | 128Mi          | 256Mi        |

Adjust in respective `deployment.yaml` files if needed.

### Scaling

```powershell
# Scale Spring Boot to 3 replicas
kubectl scale deployment/spring-boot-app -n product-app --replicas=3

# Scale React to 3 replicas
kubectl scale deployment/react-app -n product-app --replicas=3
```

### Update Database Credentials

```powershell
# Edit secret (values must be base64 encoded)
kubectl edit secret mysql-secret -n product-app

# Or delete and recreate
kubectl delete secret mysql-secret -n product-app
# Edit k8s/mysql/secret.yaml
kubectl apply -f k8s/mysql/secret.yaml

# Restart pods to pick up new credentials
kubectl rollout restart deployment/mysql -n product-app
kubectl rollout restart deployment/spring-boot-app -n product-app
```

## 🧹 Cleanup

### Delete Everything

```powershell
# Delete entire namespace (removes all resources)
kubectl delete namespace product-app
```

### Delete Specific Components

```powershell
# Delete only frontend
kubectl delete -f k8s/react/

# Delete only backend
kubectl delete -f k8s/spring-boot/

# Delete only database
kubectl delete -f k8s/mysql/
```

## 📊 Monitoring

### Watch Resources

```powershell
# Watch pods
kubectl get pods -n product-app -w

# Watch all resources
kubectl get all -n product-app -w
```

### Resource Usage

```powershell
# View resource usage (requires metrics-server)
kubectl top pods -n product-app
kubectl top nodes
```

## 🚀 Next Steps

1. **Helm Chart**: Convert these manifests to a Helm chart for easier management
2. **GitOps**: Set up ArgoCD for automated deployments
3. **Monitoring**: Add Prometheus & Grafana for observability
4. **Ingress**: Replace NodePort with Ingress for production
5. **CI/CD**: Automate image builds with GitHub Actions

## 📚 Useful Commands

```powershell
# Get all resources in namespace
kubectl get all -n product-app

# Describe a resource
kubectl describe deployment/spring-boot-app -n product-app

# Get events
kubectl get events -n product-app --sort-by='.lastTimestamp'

# Execute command in pod
kubectl exec -it -n product-app <pod-name> -- /bin/sh

# Copy files from pod
kubectl cp product-app/<pod-name>:/path/to/file ./local-file

# View YAML of running resource
kubectl get deployment/spring-boot-app -n product-app -o yaml
```

## 🔐 Security Notes

- **Secrets**: Currently using basic Kubernetes secrets. Consider using **Sealed Secrets** or **External Secrets Operator** for production
- **Network Policies**: Add network policies to restrict pod-to-pod communication
- **RBAC**: Implement proper role-based access control
- **Image Security**: Scan images with Trivy before deployment
- **Non-root Users**: Consider running containers as non-root users

## 📝 Notes

- MySQL data persists via PersistentVolumeClaim (survives pod restarts)
- Spring Boot has 2 replicas for high availability
- React app uses Nginx reverse proxy to forward `/api/*` to Spring Boot
- All services use ClusterIP except React (NodePort for external access)
- Health probes ensure pods are restarted if unhealthy
