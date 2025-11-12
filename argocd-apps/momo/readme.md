# Moment in Motion (momo) - Kubernetes Manifests

This folder contains Kubernetes manifests for deploying the Moment in Motion Django application via ArgoCD.

## Expected Manifests
- `deployment.yaml` - Application deployment
- `service.yaml` - Service exposure
- `ingress.yaml` - External access routing

## Deployment Details
- **Namespace**: `momo`
- **Image**: `ghcr.io/brar-karamjit/moment_in_motion:latest`
- **Port**: 8080

These manifests are auto-deployed by ArgoCD ApplicationSet.