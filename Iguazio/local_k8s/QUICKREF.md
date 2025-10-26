# MLRun Local Environment - Quick Reference

## 🚀 One-Time Setup

### 1. Complete Setup
```bash
cd /home/opsxe/Development/playground/Iguazio/local_k8s
make setup
```
⏱️ Takes ~15 minutes

### 2. Access Services (Choose One)

**Option A: Port Forwarding (Easiest - No DNS Setup)**
```bash
make port-forward
```
Then access at:
- MLRun UI: http://localhost:8080
- Jupyter: http://localhost:8888
- Grafana: http://localhost:3000

**Option B: DNS Setup (Optional)**
```bash
./update-windows-hosts.sh
```
Then access at http://mlrun.local:8080

---

## 🔗 Service URLs

### With Port Forwarding (localhost)
| Service | URL | 
|---------|-----|
| MLRun UI | http://localhost:8080 |
| Jupyter | http://localhost:8888 |
| MLRun API | http://localhost:8090 |
| Nuclio | http://localhost:8070 |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| MinIO | http://localhost:9001 |

### With DNS Setup (.local domains)
| Service | URL |
|---------|-----|
| MLRun UI | http://mlrun.local:8080 |
| Jupyter | http://jupyter.mlrun.local:8080 |
| Nuclio | http://nuclio.mlrun.local:8080 |
| Grafana | http://grafana.mlrun.local:8080 |

---

## 🛠️ Common Commands

### All Available Commands
```bash
make help
```

### Environment Management
```bash
make start          # Start the cluster
make stop           # Stop the cluster  
make status         # Show status
make clean          # Delete everything
```

### Access Services
```bash
make port-forward   # Forward all services to localhost
```

### Service Operations
```bash
make logs SERVICE=mlrun-ui      # View logs
make restart SERVICE=mlrun-ui   # Restart a service
make shell SERVICE=mlrun-ui     # Get shell in pod
```

### Kubectl Configuration
```bash
make kubeconfig     # Show how to set KUBECONFIG
```

---

## 🔧 Daily Workflow

### Start Working
```bash
make start
make port-forward
```
Access http://localhost:8080 in browser

### Stop Working
```bash
# Ctrl+C to stop port-forward
make stop
```

---

## �️ Cleanup

### Complete Removal
```bash
make clean
```

This will:
- Uninstall Helm releases
- Delete the k3d cluster
- Remove all data

---

## 📚 Documentation

- **Full Guide**: [README.md](README.md)
- **Makefile Help**: `make help`
- **MLRun Docs**: https://docs.mlrun.org/

---

**Environment Status**: ✅ 28/28 Pods Running
**Last Updated**: October 26, 2025
