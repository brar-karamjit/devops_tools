# Inventory Manager (invm) - Kubernetes Manifests

This folder contains Kubernetes manifests for deploying the Inventory Manager Django application via ArgoCD.

## Expected Manifests
- `deployment.yaml` - Application deployment
- `service.yaml` - Service exposure
- `ingress.yaml` - External access (optional)
- `configmap.yaml` - Configuration data
- `secret.yaml` - Sensitive data
- `pvc.yaml` - Persistent storage

## Deployment Details
- **Namespace**: `invm`
- **Image**: `ghcr.io/brar-karamjit/inventory_manager:latest`
- **Port**: 8000

These manifests are auto-deployed by ArgoCD ApplicationSet.