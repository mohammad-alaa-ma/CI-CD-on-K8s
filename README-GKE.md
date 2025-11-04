# Task Manager K8s - GKE Deployment with Cloud SQL and Helm Ingress

This document provides step-by-step instructions for deploying the Task Manager application on Google Kubernetes Engine (GKE) using Cloud SQL for database management and Helm for NGINX Ingress installation.

## Overview

This deployment showcases a cloud-native approach to running the same application demonstrated in the local CI/CD setup, but leveraging Google Cloud Platform's managed services for enhanced scalability, reliability, and reduced operational overhead.

## Architecture

```
Google Cloud Platform
├── GKE Cluster (nginx-lab)
│   ├── NGINX Ingress Controller (Helm-managed)
│   ├── Frontend Deployment (Nginx serving static files)
│   ├── Backend Deployment (Node.js API)
│   └── Documentation Deployment (Static site)
└── Cloud SQL (MySQL 8.0)
    └── app1db Database
```

## Prerequisites

- Google Cloud Platform account with billing enabled
- `gcloud` CLI installed and authenticated (`gcloud auth login`)
- `kubectl` installed and configured
- Helm installed
- Access to the application source code and Kubernetes manifests

## Step-by-Step Deployment

### 1. Create GKE Cluster

Create a small GKE cluster optimized for the task manager application:

```bash
gcloud container clusters create nginx-lab \
  --region us-central1 \
  --num-nodes 2 \
  --machine-type e2-small \
  --disk-type pd-ssd \
  --disk-size 20GB \
  --enable-ip-alias
```

Verify cluster creation:
```bash
kubectl get nodes
```

### 2. Create Cloud SQL Database

Set up a managed MySQL instance:

```bash
# Create Cloud SQL instance
gcloud sql instances create app1-mysql \
  --database-version=MYSQL_8_0 \
  --tier=db-f1-micro \
  --region=us-central1

# Create database
gcloud sql databases create app1db --instance=app1-mysql

# Set root password
gcloud sql users set-password root --host=% --instance=app1-mysql --password=<YOUR_PASSWORD>
```

### 3. Configure Database Connectivity

Choose one of the following connectivity options:

#### Option A: Public IP Access (Simple Setup)
```bash
# Get GKE node external IPs
kubectl get nodes -o wide

# Add node IPs to Cloud SQL authorized networks
gcloud sql instances patch app1-mysql \
  --authorized-networks="NODE_IP_1,NODE_IP_2"
```

#### Option B: Private IP Access (More Secure)
```bash
# Enable private IP (requires cluster in same VPC)
gcloud sql instances patch app1-mysql --assign-ip
```

### 4. Deploy Application Components

Apply your Kubernetes manifests for the application:

```bash
# Create namespace
kubectl create namespace app1

# Deploy backend
kubectl apply -f k8syamlfiles/backend.yaml -n app1

# Deploy frontend
kubectl apply -f k8syamlfiles/frontend.yaml -n app1

# Deploy documentation site
kubectl apply -f k8syamlfiles/documentaiondeployment.yaml -n app1

# Apply database secrets (update with Cloud SQL credentials)
kubectl apply -f k8syamlfiles/secretsForDBUsedByBackend.yaml -n app1
```

**Important**: Update the backend deployment to use the Cloud SQL public IP address instead of the local database IP.

**Security Best Practice**: For production deployments, it is strongly recommended to use Google Cloud Secret Manager instead of Kubernetes secrets for database credentials. This provides enhanced security, audit logging, and automatic rotation capabilities.

### 5. Install NGINX Ingress Controller via Helm

```bash
# Add Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.ingressClass=nginx \
  --set controller.watchNamespace="" \
  --set controller.admissionWebhooks.enabled=true
```

Verify installation:
```bash
kubectl get pods -n ingress-nginx
```

### 6. Configure Ingress Routing

Apply the ingress configuration to route traffic:

```bash
kubectl apply -f k8syamlfiles/ingress.yaml -n app1
```

The ingress should route:
- `app1.example.com/` → Frontend service
- `app1.example.com/api/*` → Backend service
- `nginx.example.com/` → Documentation site

### 7. Initialize Database

Connect to Cloud SQL and initialize the schema:

```bash
# Get Cloud SQL public IP
gcloud sql instances describe app1-mysql --format="value(ipAddresses.ipAddress)"

# Initialize database schema
mysql -h <CLOUD_SQL_IP> -u root -p app1db < init.sql
```

### 8. Test Database Connectivity

Verify backend can connect to Cloud SQL:

```bash
# Run a test pod
kubectl run -it --rm test-pod --image=mysql:8.0 -- bash -n app1

# Connect to database
mysql -h <CLOUD_SQL_IP> -u root -p app1db
```

### 9. Verify Deployment

Check all components are running:

```bash
# Check pods
kubectl get pods -n app1

# Check services
kubectl get svc -n app1

# Check ingress
kubectl get ingress -n app1

# Check ingress controller external IP
kubectl get svc -n ingress-nginx
```

### 10. Access the Application

Once the ingress has an external IP, update your DNS or hosts file to point the domains to the ingress IP:

- Frontend & API: `app1.example.com`
- Documentation: `nginx.example.com`

## Key Differences from Local Deployment

| Aspect | Local Deployment | GKE Deployment |
|--------|------------------|----------------|
| **Database** | External MySQL container/server | Managed Cloud SQL |
| **Ingress** | NGINX Ingress (manual install) | NGINX Ingress (Helm-managed) |
| **Cluster** | Minikube/local K8s | Managed GKE cluster |
| **Scaling** | Manual node scaling | Auto-scaling nodes |
| **Networking** | Local network access | GCP VPC networking |
| **Operations** | Self-managed | GCP-managed services |

## Cost Considerations

- **GKE Cluster**: ~$0.10/hour for e2-small nodes
- **Cloud SQL**: ~$0.015/hour for db-f1-micro
- **Ingress**: No additional cost (covered by GKE)
- **Storage**: Minimal SSD costs

Monitor costs in GCP Billing dashboard and consider cleanup when not in use.

## Cleanup

To avoid ongoing charges:

```bash
# Delete GKE cluster
gcloud container clusters delete nginx-lab --region us-central1

# Delete Cloud SQL instance
gcloud sql instances delete app1-mysql
```

## Troubleshooting

### Common Issues

1. **Database Connection Failed**
   - Verify Cloud SQL IP is correctly set in backend deployment
   - Check authorized networks include GKE node IPs
   - Ensure database credentials are correct

2. **Ingress Not Routing**
   - Verify ingress class matches controller (`nginx`)
   - Check ingress controller is running in `ingress-nginx` namespace
   - Confirm DNS points to ingress external IP

3. **Pods Not Starting**
   - Check pod logs: `kubectl logs <pod-name> -n app1`
   - Verify image pull secrets if using private registry
   - Check resource limits and node capacity

### Useful Commands

```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Debug ingress issues
kubectl describe ingress -n app1

# Check service endpoints
kubectl get endpoints -n app1

# Test connectivity from pod
kubectl exec -it <pod-name> -n app1 -- curl <service-url>
```

## CI/CD Integration

The CI/CD pipeline using Jenkins and ArgoCD for GitOps remains identical to the local deployment setup described in the main README.md. The same Jenkins pipeline (Jenkinsfile) and ArgoCD application configuration (argo-app.yaml) can be used with the GKE cluster.

### Key Points:
- **Jenkins Pipeline**: The same automated build and deployment process applies - code changes trigger builds, push images to DockerHub, and update Kubernetes manifests.
- **ArgoCD GitOps**: ArgoCD continues to monitor the `k8syamlfiles/` directory and sync changes to the GKE cluster.
- **Image Management**: Docker images are tagged with commit hashes and deployed consistently across both environments.

This ensures that the same development workflow and automation benefits are maintained in the cloud environment, providing a seamless transition from local development to production deployment.

This GKE deployment provides a production-ready, cloud-native alternative to the local development setup, demonstrating modern container orchestration with managed cloud services.