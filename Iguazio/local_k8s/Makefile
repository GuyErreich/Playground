.PHONY: help setup start stop status logs clean urls check-prereqs restart shell kubeconfig

# Variables
CLUSTER_NAME := mlrun-local
NAMESPACE := mlrun
REGISTRY_PORT := 5000
INGRESS_PORT := 8080

# Colors for output
GREEN := \033[0;32m
YELLOW := \033[1;33m
CYAN := \033[0;36m
RED := \033[0;31m
NC := \033[0m

help: ## Show this help message
	@echo "MLRun Local Development Environment"
	@echo ""
	@echo "Usage: make [target]"
	@echo ""
	@echo "Targets:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "  $(CYAN)%-20s$(NC) %s\n", $$1, $$2}'
	@echo ""

check-prereqs: ## Check if required tools are installed
	@echo "Checking prerequisites..."
	@command -v k3d >/dev/null 2>&1 || { echo "$(RED)✗ k3d not installed$(NC)"; exit 1; }
	@command -v kubectl >/dev/null 2>&1 || { echo "$(RED)✗ kubectl not installed$(NC)"; exit 1; }
	@command -v helm >/dev/null 2>&1 || { echo "$(RED)✗ helm not installed$(NC)"; exit 1; }
	@command -v docker >/dev/null 2>&1 || { echo "$(RED)✗ docker not installed$(NC)"; exit 1; }
	@echo "$(GREEN)✓ All prerequisites met$(NC)"

setup: check-prereqs ## Complete setup: create cluster, install MLRun and ingress
	@echo "$(CYAN)======================================$(NC)"
	@echo "$(CYAN) MLRun Local Setup$(NC)"
	@echo "$(CYAN)======================================$(NC)"
	@echo ""
	@echo "Step 1/4: Creating k3d cluster..."
	@if k3d cluster list | grep -q "$(CLUSTER_NAME)"; then \
		echo "$(YELLOW)Cluster already exists, skipping$(NC)"; \
	else \
		k3d cluster create $(CLUSTER_NAME) \
			--servers 1 \
			--agents 2 \
			--registry-create k3d-$(CLUSTER_NAME)-registry:0.0.0.0:$(REGISTRY_PORT) \
			--port "30040:30040@server:0" \
			--port "30050:30050@server:0" \
			--port "30060:30060@server:0" \
			--port "30070:30070@server:0" \
			--port "30080:30080@server:0" \
			--port "30090:30090@server:0" \
			--port "30010:30010@server:0" \
			--port "30020:30020@server:0" && \
		echo "$(GREEN)✓ Cluster created$(NC)"; \
	fi
	@echo ""
	@echo "Step 2/4: Setting kubectl context..."
	@k3d kubeconfig merge $(CLUSTER_NAME) --kubeconfig-merge-default
	@kubectl config use-context k3d-$(CLUSTER_NAME)
	@kubectl wait --for=condition=ready nodes --all --timeout=120s && \
		echo "$(GREEN)✓ Nodes ready$(NC)"
	@echo ""
	@echo "Step 3/4: Creating namespace and registry secret..."
	@kubectl create namespace $(NAMESPACE) --dry-run=client -o yaml | kubectl apply -f - && \
		kubectl -n $(NAMESPACE) create secret docker-registry registry-credentials \
			--docker-username=dummy \
			--docker-password=dummy \
			--docker-server=k3d-$(CLUSTER_NAME)-registry:$(REGISTRY_PORT) \
			--docker-email=dummy@example.com \
			--dry-run=client -o yaml | kubectl apply -f - && \
		echo "$(GREEN)✓ Namespace and secret created$(NC)"
	@echo ""
	@echo "Step 4/4: Installing MLRun CE (this may take 10-15 minutes)..."
	@helm repo add mlrun https://mlrun.github.io/ce >/dev/null 2>&1 || true && \
		helm repo update >/dev/null 2>&1 && \
		helm upgrade --install mlrun-ce mlrun/mlrun-ce \
			--namespace $(NAMESPACE) \
			--set global.registry.url=k3d-$(CLUSTER_NAME)-registry:$(REGISTRY_PORT) \
			--set global.registry.secretName=registry-credentials \
			--set kafka.enabled=false \
			--timeout 15m \
			--wait && \
		echo "$(GREEN)✓ MLRun CE installed$(NC)"
	@echo ""
	@echo "$(GREEN)======================================$(NC)"
	@echo "$(GREEN) Setup Complete! 🎉$(NC)"
	@echo "$(GREEN)======================================$(NC)"
	@echo ""
	@echo "$(CYAN)Access services via NodePort:$(NC)"
	@echo "  Jupyter:         http://localhost:30040"
	@echo "  Nuclio:          http://localhost:30050"
	@echo "  MLRun UI:        http://localhost:30060"
	@echo "  MLRun API:       http://localhost:30070"
	@echo "  MinIO API:       http://localhost:30080"
	@echo "  MinIO Console:   http://localhost:30090"
	@echo "  Grafana:         http://localhost:30010"
	@echo "  Prometheus:      http://localhost:30020"
	@echo ""
	@echo "$(CYAN)To see all service URLs:$(NC)"
	@echo "  make urls"
	@echo ""
	@echo "$(CYAN)Set kubectl context:$(NC)"
	@echo "  kubectl config use-context k3d-$(CLUSTER_NAME)"
	@echo ""
	@echo "$(CYAN)Or export KUBECONFIG directly:$(NC)"
	@echo "  export KUBECONFIG=\"$$HOME/.config/k3d/kubeconfig-$(CLUSTER_NAME).yaml\""
	@echo ""

urls: ## Show all service URLs
	@echo "$(CYAN)======================================$(NC)"
	@echo "$(CYAN) MLRun Service URLs$(NC)"
	@echo "$(CYAN)======================================$(NC)"
	@echo ""
	@echo "$(GREEN)Primary Services:$(NC)"
	@echo "  $(CYAN)Jupyter Notebook:$(NC)  http://localhost:30040"
	@echo "  $(CYAN)Nuclio Dashboard:$(NC)  http://localhost:30050"
	@echo "  $(CYAN)MLRun UI:$(NC)          http://localhost:30060"
	@echo "  $(CYAN)MLRun API:$(NC)         http://localhost:30070"
	@echo ""
	@echo "$(GREEN)Storage:$(NC)"
	@echo "  $(CYAN)MinIO API:$(NC)         http://localhost:30080"
	@echo "  $(CYAN)MinIO Console:$(NC)     http://localhost:30090"
	@echo "    $(YELLOW)(user: minio, pass: minio123)$(NC)"
	@echo ""
	@echo "$(GREEN)Monitoring:$(NC)"
	@echo "  $(CYAN)Grafana:$(NC)           http://localhost:30010"
	@echo "  $(CYAN)Prometheus:$(NC)        http://localhost:30020"
	@echo ""
	@echo "$(YELLOW)Note: Ports are mapped from k3d cluster to localhost$(NC)"
	@echo ""

start: ## Start the k3d cluster
	@echo "Starting k3d cluster..."
	@k3d cluster start $(CLUSTER_NAME)
	@kubectl config use-context k3d-$(CLUSTER_NAME)
	@echo "$(GREEN)✓ Cluster started$(NC)"
	@echo ""
	@echo "Waiting for pods to be ready (this may take 2-3 minutes)..."
	@sleep 5
	@kubectl -n $(NAMESPACE) get pods
	@echo ""
	@echo "$(CYAN)Service URLs:$(NC) make urls"

stop: ## Stop the k3d cluster
	@echo "Stopping k3d cluster..."
	@k3d cluster stop $(CLUSTER_NAME)
	@echo "$(GREEN)✓ Cluster stopped$(NC)"

status: ## Show cluster and pod status
	@echo "$(CYAN)Cluster Status:$(NC)"
	@k3d cluster list | grep $(CLUSTER_NAME) || echo "$(YELLOW)Cluster not found$(NC)"
	@echo ""
	@echo "$(CYAN)Node Status:$(NC)"
	@kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true
	@kubectl get nodes 2>/dev/null || echo "$(YELLOW)Cluster not running$(NC)"
	@echo ""
	@echo "$(CYAN)Pod Status:$(NC)"
	@kubectl -n $(NAMESPACE) get pods 2>/dev/null || echo "$(YELLOW)Cluster not running$(NC)"
	@echo ""
	@RUNNING=$$(kubectl -n $(NAMESPACE) get pods --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l) && \
		TOTAL=$$(kubectl -n $(NAMESPACE) get pods --no-headers 2>/dev/null | wc -l) && \
		echo "$(GREEN)Running pods: $$RUNNING / $$TOTAL$(NC)"

logs: ## Show logs for a service (usage: make logs SERVICE=mlrun-ui)
	@if [ -z "$(SERVICE)" ]; then \
		echo "$(RED)Error: SERVICE not specified$(NC)"; \
		echo "Usage: make logs SERVICE=<service-name>"; \
		echo ""; \
		echo "Available services:"; \
		kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true && \
			kubectl -n $(NAMESPACE) get deployments --no-headers | awk '{print "  - " $$1}'; \
		exit 1; \
	fi
	@kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true
	@kubectl -n $(NAMESPACE) logs -f deployment/$(SERVICE)

clean: ## Delete the cluster and all data
	@echo "$(YELLOW)Warning: This will delete the entire cluster and all data!$(NC)"
	@read -p "Are you sure? (yes/no): " answer && [ "$$answer" = "yes" ] || exit 1
	@echo ""
	@echo "Uninstalling Helm releases..."
	@kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true
	@helm uninstall mlrun-ce -n $(NAMESPACE) 2>/dev/null || true
	@echo ""
	@echo "Deleting k3d cluster..."
	@k3d cluster delete $(CLUSTER_NAME)
	@echo ""
	@echo "$(GREEN)✓ Cleanup complete$(NC)"
	@echo ""
	@echo "$(YELLOW)Note: Remove DNS entries from Windows hosts file if needed$(NC)"

restart: ## Restart a specific service (usage: make restart SERVICE=mlrun-ui)
	@if [ -z "$(SERVICE)" ]; then \
		echo "$(RED)Error: SERVICE not specified$(NC)"; \
		echo "Usage: make restart SERVICE=<service-name>"; \
		exit 1; \
	fi
	@kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true
	@kubectl -n $(NAMESPACE) rollout restart deployment/$(SERVICE) && \
		echo "$(GREEN)✓ Restarted $(SERVICE)$(NC)"

shell: ## Get a shell in a pod (usage: make shell SERVICE=mlrun-ui)
	@if [ -z "$(SERVICE)" ]; then \
		echo "$(RED)Error: SERVICE not specified$(NC)"; \
		echo "Usage: make shell SERVICE=<service-name>"; \
		exit 1; \
	fi
	@kubectl config use-context k3d-$(CLUSTER_NAME) 2>/dev/null || true
	@POD=$$(kubectl -n $(NAMESPACE) get pods -l app=$(SERVICE) -o jsonpath='{.items[0].metadata.name}' 2>/dev/null) && \
		if [ -z "$$POD" ]; then \
			POD=$$(kubectl -n $(NAMESPACE) get pods | grep $(SERVICE) | head -1 | awk '{print $$1}'); \
		fi && \
		kubectl -n $(NAMESPACE) exec -it $$POD -- /bin/bash || \
		kubectl -n $(NAMESPACE) exec -it $$POD -- /bin/sh

kubeconfig: ## Print the kubectl config command to use
	@echo "$(CYAN)Add this to your shell:$(NC)"
	@echo ""
	@echo "export KUBECONFIG=\"$$HOME/.config/k3d/kubeconfig-$(CLUSTER_NAME).yaml\""
	@echo ""
	@echo "$(CYAN)Or add to your ~/.zshrc or ~/.bashrc:$(NC)"
	@echo ""
	@echo "echo 'export KUBECONFIG=\"\$$HOME/.config/k3d/kubeconfig-$(CLUSTER_NAME).yaml\"' >> ~/.zshrc"
