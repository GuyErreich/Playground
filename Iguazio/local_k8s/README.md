# MLRun Local Development Environment

Complete MLRun CE setup with k3d, NGINX Ingress, and DNS-based access from Windows.

> 💡 **Quick Start**: See [QUICKREF.md](QUICKREF.md) for a one-page reference with common commands.

## 📖 Table of Contents

- [Quick Access](#-quick-access)
- [Step-by-Step Setup Guide](#-step-by-step-setup-guide)
- [Architecture](#-architecture)
- [What's Deployed](#-whats-deployed)
- [Management Commands](#️-management-commands)
- [Troubleshooting](#-troubleshooting)

## �🚀 Quick Access

Once setup is complete, access services from Windows Edge:

- **MLRun UI**: http://mlrun.local:8080
- **Jupyter Notebook**: http://jupyter.mlrun.local:8080
- **Nuclio Dashboard**: http://nuclio.mlrun.local:8080
- **MLRun API**: http://api.mlrun.local:8080
- **Grafana**: http://grafana.mlrun.local:8080
- **Prometheus**: http://prometheus.mlrun.local:8080
- **MinIO Console**: http://minio.mlrun.local:8080
  - Username: `minio`
  - Password: `minio123`

## 🏗️ Step-by-Step Setup Guide

Follow these steps to set up the MLRun local development environment from scratch.

### 🚀 Quick Setup (Automated)

For automated setup, simply run:

```bash
cd /home/opsxe/Development/playground/Iguazio/local_k8s
./setup.sh
```

This script will:
1. Create the k3d cluster with registry
2. Install NGINX Ingress Controller
3. Install MLRun CE with all components
4. Configure ingress rules
5. Display access information

**Then follow the on-screen instructions to configure Windows hosts file.**

---

### 📝 Manual Setup (Step-by-Step)

For manual setup or to understand each step:

### Prerequisites

Ensure you have the following installed in WSL2:
- Docker Desktop (with WSL2 integration enabled)
- k3d (v5.8.3 or later)
- kubectl
- helm (v3.6 or later)

### Step 1: Create the k3d Cluster

Create a k3d cluster with 1 server and 2 agent nodes, plus a local Docker registry:

```bash
# Create the cluster with local registry
k3d cluster create mlrun-local \
  --servers 1 \
  --agents 2 \
  --registry-create k3d-mlrun-local-registry:0.0.0.0:5000 \
  --port "8080:30242@server:0"

# Verify cluster is running
k3d cluster list

# Set kubectl context
# Option A: use the kubeconfig file that k3d created (recommended)
# k3d writes a kubeconfig at: $HOME/.config/k3d/kubeconfig-<cluster-name>.yaml
export KUBECONFIG="$HOME/.config/k3d/kubeconfig-mlrun-local.yaml"

# Verify kubectl is pointing at the new cluster
kubectl config current-context
kubectl get nodes

# Persisting KUBECONFIG (optional): add to your shell rc to make the context permanent
# bash/zsh:
echo 'export KUBECONFIG="$HOME/.config/k3d/kubeconfig-mlrun-local.yaml"' >> "$HOME/.bashrc"
# or for zsh:
echo 'export KUBECONFIG="$HOME/.config/k3d/kubeconfig-mlrun-local.yaml"' >> "$HOME/.zshrc"

# Option B: use kubectx (convenient context switcher)
# Install (one of):
#  - Debian/Ubuntu: sudo apt install kubectx    (may not be available on all distros)
#  - Homebrew (recommended on macOS/WSL with Linuxbrew): brew install kubectx
#  - Manual: git clone https://github.com/ahmetb/kubectx /opt/kubectx && sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx && sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens
#
# Usage:
# List contexts:
kubectx
# Switch context to the k3d cluster:
kubectx k3d-mlrun-local
# Switch namespace interactively with kubens:
kubens mlrun

# Note: `kubectl config use-context <name>` is equivalent to `kubectx <name>`; kubectx just makes it faster to list/switch.
```

**Expected output:**
```
NAME                       STATUS   ROLES                  AGE
k3d-mlrun-local-server-0   Ready    control-plane,master   1m
k3d-mlrun-local-agent-0    Ready    <none>                 1m
k3d-mlrun-local-agent-1    Ready    <none>                 1m
```

> **Note**: The `--port "8080:30242@server:0"` maps host port 8080 to the NGINX Ingress NodePort (30242), enabling DNS-based access from Windows.

### Step 2: Create MLRun Namespace and Registry Secret

```bash
# Create the mlrun namespace
kubectl create namespace mlrun

# Create a dummy Docker registry secret (required by MLRun CE)
kubectl -n mlrun create secret docker-registry registry-credentials \
  --docker-username=dummy \
  --docker-password=dummy \
  --docker-server=k3d-mlrun-local-registry:5000 \
  --docker-email=dummy@example.com
```

### Step 3: Install NGINX Ingress Controller

```bash
# Add the ingress-nginx Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.hostPort.enabled=false \
  --set controller.service.type=NodePort \
  --set controller.config.use-forwarded-headers="true"

# Wait for the controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

**Verify NGINX Ingress is running:**
```bash
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc
```

You should see the ingress-nginx-controller pod running and the service with NodePort (typically 30242).

### Step 4: Install MLRun CE Chart

```bash
# Add the MLRun Helm repository
helm repo add mlrun https://mlrun.github.io/ce
helm repo update

# Install MLRun CE with default configuration
helm install mlrun-ce mlrun/mlrun-ce \
  --namespace mlrun \
  --set global.registry.url=k3d-mlrun-local-registry:5000 \
  --set global.registry.secretName=registry-credentials \
  --set kafka.enabled=false \
  --timeout 15m \
  --wait

# Monitor the deployment
watch kubectl -n mlrun get pods
```

**Wait for all pods to be Running** (this can take 5-10 minutes). Press Ctrl+C to exit watch when done.

**Expected result**: All 28 pods in Running state:
```bash
kubectl -n mlrun get pods --field-selector=status.phase=Running --no-headers | wc -l
# Should output: 28
```

> **Note**: Kafka is disabled (`--set kafka.enabled=false`) due to a non-existent image tag issue in MLRun CE 0.9.0.

### Step 5: Create Ingress Rules

Create an ingress resource to route traffic based on hostname:

```bash
# Apply the ingress configuration
kubectl apply -f mlrun-ingress.yaml

# Verify ingress is created
kubectl -n mlrun get ingress
```

**Expected output:**
```
NAME            CLASS   HOSTS                                                         ADDRESS
mlrun-ingress   nginx   mlrun.local,api.mlrun.local,jupyter.mlrun.local + 4 more...   10.43.x.x
```

### Step 6: Configure Windows Hosts File

Now configure Windows to resolve the MLRun DNS names.

#### Option A: From WSL (Bash Script)

**Note**: This requires the Windows hosts file to be writable from WSL. If you get a permission error, use Option B instead.

```bash
# Run the bash script from WSL
./update-windows-hosts.sh
```

If you get a permission error, see the script output for instructions, or use Option B below.

#### Option B: From Windows PowerShell (Recommended)

1. Open **PowerShell as Administrator** (right-click → "Run as Administrator")
2. Navigate to the project directory:
   ```powershell
   cd \\wsl$\Ubuntu\home\opsxe\Development\playground\Iguazio\local_k8s
   ```
3. Run the PowerShell update script:
   ```powershell
   .\update-windows-hosts.ps1
   ```

#### Option C: Manual

1. Open **Notepad as Administrator**
2. Open file: `C:\Windows\System32\drivers\etc\hosts`
3. Add these lines at the end:
   ```
   # MLRun Local Development
   127.0.0.1 mlrun.local
   127.0.0.1 api.mlrun.local
   127.0.0.1 jupyter.mlrun.local
   127.0.0.1 nuclio.mlrun.local
   127.0.0.1 minio.mlrun.local
   127.0.0.1 grafana.mlrun.local
   127.0.0.1 prometheus.mlrun.local
   ```
4. Save the file

### Step 7: Test Access from Windows

1. Open **Microsoft Edge** on Windows
2. Navigate to: **http://mlrun.local:8080**
3. You should see the MLRun UI! 🎉

Try other services:
- Jupyter: http://jupyter.mlrun.local:8080
- Nuclio: http://nuclio.mlrun.local:8080
- Grafana: http://grafana.mlrun.local:8080

### Step 8: Verify All Components (Optional)

From WSL terminal, run these verification commands:

```bash
# Check all pods are running
kubectl -n mlrun get pods

# Check all services
kubectl -n mlrun get svc

# Check ingress configuration
kubectl -n mlrun get ingress

# Test ingress routing from WSL
curl -s -H "Host: mlrun.local" http://localhost:8080 | head -5
```

### 🎊 Setup Complete!

Your MLRun local development environment is ready to use. All 28 components are running and accessible via friendly DNS names from your Windows browser.

## 📋 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows Host                              │
│                                                              │
│  ┌──────────────────┐         ┌──────────────────────┐     │
│  │ Microsoft Edge   │         │ Windows hosts file   │     │
│  │                  │         │ 127.0.0.1 mlrun.local│     │
│  │ mlrun.local:8080 │────────▶│ 127.0.0.1 *.local    │     │
│  └──────────────────┘         └──────────────────────┘     │
│           │                                                  │
│           │ localhost:8080                                   │
└───────────┼──────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│                      WSL2 / Docker                           │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           k3d Cluster: mlrun-local                    │  │
│  │                                                       │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │  k3d LoadBalancer (serverlb)                 │   │  │
│  │  │  Port Mapping: 8080 → 30242                  │   │  │
│  │  └──────────────────┬───────────────────────────┘   │  │
│  │                     │                                 │  │
│  │                     ▼                                 │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │  NGINX Ingress Controller                    │   │  │
│  │  │  Namespace: ingress-nginx                    │   │  │
│  │  │  NodePort: 30242                             │   │  │
│  │  │                                               │   │  │
│  │  │  Host-based routing:                         │   │  │
│  │  │  • mlrun.local → mlrun-ui:80                 │   │  │
│  │  │  • jupyter.mlrun.local → mlrun-jupyter:8888  │   │  │
│  │  │  • nuclio.mlrun.local → nuclio-dashboard     │   │  │
│  │  │  • grafana.mlrun.local → grafana:80          │   │  │
│  │  └──────────────────┬───────────────────────────┘   │  │
│  │                     │                                 │  │
│  │                     ▼                                 │  │
│  │  ┌──────────────────────────────────────────────┐   │  │
│  │  │        MLRun Services (Namespace: mlrun)     │   │  │
│  │  │                                               │   │  │
│  │  │  Core:                    Monitoring:         │   │  │
│  │  │  • mlrun-api (8080)       • Prometheus       │   │  │
│  │  │  • mlrun-ui (80)          • Grafana          │   │  │
│  │  │  • mlrun-db (MySQL)       • Node Exporter    │   │  │
│  │  │  • mlrun-jupyter (8888)                      │   │  │
│  │  │                                               │   │  │
│  │  │  Serverless:              Storage:            │   │  │
│  │  │  • Nuclio Dashboard       • MinIO (S3)       │   │  │
│  │  │  • Nuclio Controller      • TDEngine (TSDB)  │   │  │
│  │  │                                               │   │  │
│  │  │  ML Operators:            Pipelines:          │   │  │
│  │  │  • Spark Operator         • KFP Metadata     │   │  │
│  │  │  • MPI Operator           • ML Pipeline      │   │  │
│  │  │                           • Visualization     │   │  │
│  │  │                                               │   │  │
│  │  │  Registry:                                    │   │  │
│  │  │  • k3d-mlrun-local-registry:5000             │   │  │
│  │  └──────────────────────────────────────────────┘   │  │
│  │                                                       │  │
│  │  Nodes: 1 server + 2 agents                          │  │
│  │  Total Pods: 28                                      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Traffic Flow:**
1. User browses to `http://mlrun.local:8080` in Edge
2. Windows hosts file resolves `mlrun.local` → `127.0.0.1`
3. Request hits k3d LoadBalancer on port `8080`
4. LoadBalancer forwards to NGINX Ingress NodePort `30242`
5. NGINX Ingress routes based on hostname to correct service
6. Service responds back through the chain

## ✅ What's Deployed

### Core MLRun Stack
- **MLRun API** (Chief + Workers)
- **MLRun UI** (Web interface)
- **MLRun DB** (MySQL)
- **Jupyter Notebook** (with MLRun integration)

### Serverless & ML
- **Nuclio** (Controller + Dashboard)
- **Spark Operator** (Controller + Webhook)
- **MPI Operator** (for distributed training)

### Storage & Data
- **MinIO** (S3-compatible object storage)
- **TDEngine** (Time-series database)

### Monitoring Stack
- **Prometheus** (Metrics collection)
- **Grafana** (Dashboards)
- **Node Exporter** (System metrics)
- **Prometheus Operator**
- **Kube State Metrics**

### ML Pipelines
- **Kubeflow Pipelines** (ML Pipeline, Metadata, Visualization, Scheduled Workflows)
- **MySQL** (Pipeline storage)

### Networking
- **NGINX Ingress Controller** (HTTP routing)

**Total: 28 Pods Running**

> **Note**: Kafka was disabled due to a non-existent image tag issue in the MLRun CE 0.9.0 chart.

## 🔧 Additional Configuration (Optional)

### Exposing Additional Ports

If you need to expose more services directly from Windows, you can add more port mappings:

```bash
# Edit cluster to add more port mappings
k3d cluster edit mlrun-local --port-add "9090:30020@server:0"  # Prometheus
k3d cluster edit mlrun-local --port-add "3000:30010@server:0"  # Grafana
```

Then access directly:
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000

### Configuring Persistent Storage

The default setup uses k3d's built-in local-path-provisioner. For production-like persistence:

1. Check existing PVCs:
   ```bash
   kubectl -n mlrun get pvc
   ```

2. Configure storage class:
   ```bash
   kubectl patch storageclass local-path \
     -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

### Enabling Kafka (Advanced)

If you need Kafka and can resolve the image issue:

```bash
# Update MLRun CE with a working Kafka image
helm upgrade mlrun-ce mlrun/mlrun-ce \
  --namespace mlrun \
  --reuse-values \
  --set kafka.enabled=true \
  --set kafka.image.tag=3.8.1-debian-12-r0
```

## 🔄 Starting and Stopping the Environment

### Starting the Cluster

If you've stopped the cluster and want to restart it:

```bash
# Start the k3d cluster
k3d cluster start mlrun-local

# Verify all pods are running (may take 2-3 minutes)
kubectl -n mlrun get pods

# Check services are ready
kubectl -n mlrun get svc
```

### Stopping the Cluster

To free up resources when not using MLRun:

```bash
# Stop the k3d cluster (preserves all data)
k3d cluster stop mlrun-local
```

### Removing the Environment

To completely remove the cluster and all data:

```bash
# Use the automated cleanup script
./cleanup.sh

# Or manually delete
k3d cluster delete mlrun-local

# Remove the registry
docker volume rm k3d-mlrun-local-registry

# Clean up dangling volumes (optional)
docker volume prune
```

**Don't forget to remove entries from Windows hosts file** (`C:\Windows\System32\drivers\etc\hosts`) if you added them.

## 📁 Project Structure

```
local_k8s/
├── README.md                      # This file - complete documentation
├── setup.sh                       # Automated setup script
├── cleanup.sh                     # Automated cleanup script
├── mlrun-ingress.yaml            # Ingress configuration for MLRun services
├── update-windows-hosts.ps1      # PowerShell script to update Windows hosts file
├── terraform/                     # Terraform infrastructure code (optional)
│   ├── main.tf
│   ├── cluster.tf
│   ├── variables.tf
│   └── outputs.tf
└── mlrun-local/                   # Custom Helm chart (deprecated - not used)
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

## 🛠️ Management Commands

### Viewing Logs

```bash
# MLRun API logs
kubectl -n mlrun logs -f deployment/mlrun-api-chief

# MLRun UI logs
kubectl -n mlrun logs -f deployment/mlrun-ui

# Jupyter logs
kubectl -n mlrun logs -f deployment/mlrun-jupyter

# Nuclio dashboard logs
kubectl -n mlrun logs -f deployment/nuclio-dashboard
```

### Port Forwarding (Alternative Access)

If ingress isn't working, you can port-forward directly:

```bash
# MLRun UI
kubectl -n mlrun port-forward svc/mlrun-ui 8080:80

# Jupyter
kubectl -n mlrun port-forward svc/mlrun-jupyter 8888:8888
```

Then access via http://localhost:8080 or http://localhost:8888

## 🔄 Updating Components

### Update MLRun CE

```bash
helm repo update
helm upgrade mlrun-ce mlrun/mlrun-ce --namespace mlrun --reuse-values
```

### Update NGINX Ingress

```bash
helm repo update
helm upgrade ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values
```

### Update Ingress Rules

Edit `mlrun-ingress.yaml` and apply:

```bash
kubectl apply -f mlrun-ingress.yaml
```

## 🐛 Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl -n mlrun get pods

# Describe problematic pod
kubectl -n mlrun describe pod <pod-name>

# Check events
kubectl -n mlrun get events --sort-by='.lastTimestamp'
```

### Ingress Not Working

```bash
# Check ingress status
kubectl -n mlrun get ingress

# Check NGINX Ingress logs
kubectl -n ingress-nginx logs -f deployment/ingress-nginx-controller

# Verify port mapping
k3d cluster list mlrun-local
```

### DNS Not Resolving

1. Verify Windows hosts file has entries (open as Administrator)
2. Flush DNS cache:
   ```powershell
   ipconfig /flushdns
   ```
3. Restart browser

### Services Unreachable from Windows

1. Check WSL IP is accessible:
   ```bash
   # In WSL
   hostname -I
   ```
2. Try direct NodePort access:
   - http://172.18.10.35:30060 (MLRun UI)
3. Check Windows Firewall isn't blocking WSL

## 📊 Resource Usage

Expected resource consumption:
- **CPU**: ~2-4 cores
- **Memory**: ~6-8 GB
- **Disk**: ~10 GB (including images)

## 🔐 Security Notes

⚠️ **This is a development environment only!**

- Default passwords are used (e.g., MinIO: minio/minio123)
- No authentication on MLRun API
- No TLS/HTTPS
- Services exposed on host network

**DO NOT use in production!**

## 📚 Additional Resources

- [MLRun Documentation](https://docs.mlrun.org/)
- [MLRun GitHub](https://github.com/mlrun/mlrun)
- [Nuclio Documentation](https://nuclio.io/docs/)
- [k3d Documentation](https://k3d.io/)

## 🔗 Service Endpoints Summary

| Service | URL | NodePort (Direct) |
|---------|-----|-------------------|
| MLRun UI | http://mlrun.local:8080 | http://172.18.10.35:30060 |
| Jupyter | http://jupyter.mlrun.local:8080 | http://172.18.10.35:30040 |
| Nuclio | http://nuclio.mlrun.local:8080 | http://172.18.10.35:30050 |
| MLRun API | http://api.mlrun.local:8080 | http://172.18.10.35:30070 |
| Grafana | http://grafana.mlrun.local:8080 | http://172.18.10.35:30010 |
| Prometheus | http://prometheus.mlrun.local:8080 | http://172.18.10.35:30020 |
| MinIO | http://minio.mlrun.local:8080 | http://172.18.10.35:30090 |

---

**Status**: ✅ All systems operational (28/28 pods running)
