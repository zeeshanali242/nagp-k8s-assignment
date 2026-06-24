# NAGP 2026 - Kubernetes, DevOps & FinOps Assignment

## Project Overview

A multi-tier application deployed on **Google Kubernetes Engine (GKE)** featuring a **Spring Boot REST API** (Service Tier) connected to a **MySQL database** (Database Tier). The API exposes employee records and demonstrates Kubernetes concepts including rolling updates, self-healing, persistent storage, HPA, ConfigMaps, Secrets, and Ingress.

---

## Links

| Resource | URL |
|----------|-----|
| **Code Repository** | https://github.com/zeeshanali242/nagp-k8s-assignment.git |
| **Docker Hub Image** | `https://hub.docker.com/r/zeeshanali242/k8s-assignment-api` |
| **API Endpoint** | `http://8.233.196.9/api/employees` |
| **Screen Recording** | [Watch Recording](https://nagarro-my.sharepoint.com/:v:/p/mohd_ali03/IQDIT9dVqSA_T6M2M6mbaqQjAXN6GJ6gt-4w3ACMaZhpPrU) |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Service API Tier | Spring Boot 3.2.5 (Java 17) |
| Database Tier | MySQL 8.0 |
| Container Runtime | Docker |
| Orchestration | Google Kubernetes Engine (GKE) |
| Ingress Controller | GCE (Google Cloud Load Balancer) |
| Cloud Platform | Google Cloud Platform (GCP) |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   Google Kubernetes Engine (GKE)                   │
│                                                                    │
│  ┌───────────────┐     ┌──────────────────────────────────────┐  │
│  │ GCE Ingress   │────▶│  API Service (NodePort, port 80)     │  │
│  │ (Cloud LB)    │     └──────────────────────────────────────┘  │
│  └───────────────┘                     │                          │
│       ▲                                ▼                          │
│       │                  ┌───────────────────────┐                │
│   External               │   API Deployment      │                │
│   Traffic                │   (4 replicas + HPA)  │                │
│   (Public IP)            └───────────────────────┘                │
│                                        │                          │
│                                        ▼                          │
│                          ┌───────────────────────┐                │
│                          │ MySQL Service          │                │
│                          │ (ClusterIP - internal) │                │
│                          └───────────────────────┘                │
│                                        │                          │
│                                        ▼                          │
│                          ┌───────────────────────┐                │
│                          │ MySQL Deployment      │                │
│                          │ (1 replica + PVC)     │                │
│                          └───────────────────────┘                │
│                                        │                          │
│                                        ▼                          │
│                          ┌───────────────────────┐                │
│                          │ PersistentVolumeClaim │                │
│                          │ (10Gi GCP PD)         │                │
│                          └───────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- Google Cloud SDK (`gcloud`) installed and configured
- Docker & Docker Hub account
- `kubectl` CLI
- GCP project with billing enabled

---

## Quick Start - Deploy to GCP (GKE)

### 0. Clone the Repository

```bash
git clone https://github.com/zeeshanali242/nagp-k8s-assignment.git
cd nagp-k8s-assignment
```

### 1. Set Up GCP Project

```bash
# Authenticate with GCP
gcloud auth login

# Set your project
export GCP_PROJECT_ID="your-gcp-project-id"
gcloud config set project $GCP_PROJECT_ID

# Enable required APIs
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
```

### 2. Create GKE Cluster

```bash
gcloud container clusters create nagp-cluster \
    --zone us-east1-b \
    --num-nodes 2 \
    --machine-type e2-medium \
    --disk-size 30 \
    --enable-autoscaling --min-nodes 1 --max-nodes 3 \
    --enable-ip-alias

# Get cluster credentials
gcloud container clusters get-credentials nagp-cluster --zone us-east1-b
```

### 3. Build & Push Docker Image

```bash
# Build the image
docker build -t zeeshanali242/k8s-assignment-api:1.1.0 .

# Push to Docker Hub
docker login
docker push zeeshanali242/k8s-assignment-api:1.1.0
```

### 4. Deploy to GKE with `kubectl`

The app is deployed via the plain Kubernetes manifests in [`k8s/`](k8s). Apply them in
dependency order. The DB password is stored as a base64 Kubernetes **Secret**
(`k8s/secret.yaml`).

```bash
# Namespace first
kubectl apply -f k8s/namespace.yaml

# Config + Secret
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml

# Database tier
kubectl apply -f k8s/mysql-pvc.yaml
kubectl apply -f k8s/mysql-deployment.yaml
kubectl apply -f k8s/mysql-service.yaml

# Service / API tier
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/api-ingress.yaml
kubectl apply -f k8s/api-hpa.yaml

# Or apply everything at once
kubectl apply -f k8s/

# Wait for the API rollout
kubectl rollout status deployment/api-service -n nagp-assignment
```

To **rotate the password** later, update the base64 values in `k8s/secret.yaml` and
re-apply (then restart the pods so they pick up the new value):

```bash
kubectl apply -f k8s/secret.yaml
kubectl rollout restart deployment/api-service deployment/mysql -n nagp-assignment
```

**For a full step-by-step demo runbook (with verification at each step), see [`COMMANDS.md`](doc/COMMANDS.md).**


### 5. Get External IP

GCE Ingress provisions a Cloud Load Balancer with a public IP (takes 2-5 minutes):

```bash
# Check Ingress status
kubectl get ingress api-ingress -n nagp-assignment

# Wait and watch for the IP
kubectl get ingress api-ingress -n nagp-assignment -w
```

### 6. Test the API

```bash
# Get the external IP
EXTERNAL_IP=$(kubectl get ingress api-ingress -n nagp-assignment -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Bulk-insert 10 employee records (one call)
curl -X POST http://$EXTERNAL_IP/api/employees/bulk \
  -H "Content-Type: application/json" \
  --data-binary @postman/bulk-employees.json

# Get all employees
curl http://$EXTERNAL_IP/api/employees

# Get single employee
curl http://$EXTERNAL_IP/api/employees/1

# Health check
curl http://$EXTERNAL_IP/actuator/health
```

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/employees` | List all employees |
| `GET` | `/api/employees/{id}` | Get one employee |
| `POST` | `/api/employees` | Create a single employee |
| `POST` | `/api/employees/bulk` | Bulk-create employees (JSON array) |
| `GET` | `/api/health`, `/actuator/health` | Health checks |

> **Postman:** import `postman/NAGP-Employee-API.postman_collection.json`, set the
> `baseUrl` variable to `http://<INGRESS_IP>`, and run **"Bulk Create Employees (10 records)"**.

---

## Local Development (Docker Compose)

```bash
# Start all services
docker-compose up -d

# Test
curl http://localhost:8080/api/employees

# Stop
docker-compose down
```

---

## Kubernetes Features Demonstrated

| Feature | How |
|---------|-----|
| **Rolling Updates** | API deployment uses `RollingUpdate` strategy (maxSurge=1, maxUnavailable=1) |
| **Self-Healing** | Kubernetes restarts failed pods automatically via liveness/readiness probes |
| **Persistent Storage** | MySQL uses PVC backed by GCP Persistent Disk - data survives pod restarts |
| **ConfigMap** | DB connection URL and username externalized |
| **Secrets** | DB password stored as K8s Secret (base64 encoded) |
| **HPA** | API auto-scales 4→8 replicas based on CPU (70%) and memory (80%) |
| **Ingress** | GCE Ingress exposes API externally via Cloud Load Balancer |
| **Internal-only DB** | MySQL service is ClusterIP (no external access) |

---

## Demonstration Commands

```bash
# View all objects
kubectl get all -n nagp-assignment

# Show self-healing - kill an API pod
kubectl delete pod -l app=api-service -n nagp-assignment --wait=false
kubectl get pods -n nagp-assignment -w

# Show DB persistence - kill MySQL pod
kubectl delete pod -l app=mysql -n nagp-assignment --wait=false
kubectl get pods -n nagp-assignment -w
# After MySQL restarts, verify data is still there:
curl http://$EXTERNAL_IP/api/employees

# Show rolling update
kubectl set image deployment/api-service api-service=zeeshanali242/k8s-assignment-api:1.2.0 -n nagp-assignment
kubectl rollout status deployment/api-service -n nagp-assignment

# Show HPA
kubectl get hpa -n nagp-assignment

# Show resource usage
kubectl top pods -n nagp-assignment
```

---

## FinOps Considerations

### Resource Requests & Limits

| Tier | CPU Request | CPU Limit | Memory Request | Memory Limit |
|------|-------------|-----------|----------------|--------------|
| API Service | 100m | 500m | 256Mi | 512Mi |
| MySQL | 250m | 500m | 512Mi | 1Gi |

### Three Cost Optimization Opportunities

1. **Right-sizing based on observed metrics**: After running `kubectl top pods`, adjust resource requests/limits to match actual usage. Over-provisioned resources waste money on unused capacity.

2. **Use GKE Spot VMs for stateless workloads**: For the stateless API tier, use Spot VMs (up to 91% cheaper on GCP). Only the database tier needs on-demand instances for reliability. Configure node pools with `--spot` flag.

3. **GKE Cluster Autoscaler + Off-peak scaling**: Use GKE's built-in cluster autoscaler to remove underutilized nodes. Combine with scheduled scaling (CronJobs or KEDA) to reduce HPA `minReplicas` during off-peak hours.
---

## Project Structure

```
nagp-k8s-assignment/
├── src/main/java/com/nagp/k8sassignment/
│   ├── K8sAssignmentApplication.java
│   ├── controller/EmployeeController.java
│   ├── model/Employee.java
│   └── repository/EmployeeRepository.java
├── src/main/resources/
│   └── application.yml
├── k8s/                          # Plain Kubernetes manifests (deploy with kubectl)
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml               # base64 DB password Secret
│   ├── mysql-pvc.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── api-ingress.yaml
│   └── api-hpa.yaml
├── Dockerfile
├── docker-compose.yaml
├── postman/
│   ├── NAGP-Employee-API.postman_collection.json
│   └── bulk-employees.json
├── pom.xml
├── README.md
```
