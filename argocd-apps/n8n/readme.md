# n8n app

This folder contains Kubernetes manifests for deploying n8n with a PostgreSQL backend managed in the `db` namespace.

ArgoCD auto-discovers this folder via `argocd/applicationset-all.yaml` and creates the `n8n-app` Application in namespace `n8n`.

## Manifests
- `deployment.yaml`: Deployment, Service, and Ingress
- `externalsecret.yaml`: syncs credentials from Bitwarden to `n8n-secrets`
- `istio.yaml`: namespace manifest with Istio injection disabled

## Required Bitwarden secrets
- `n8n-db-username`
- `n8n-db-password`
- `n8n-encryption-key`

## Expected database endpoint
- Host: `n8n-db.db.svc.cluster.local`
- Port: `5432`
- Database: `n8n`

## External URL
- `https://n8n.karamjitbrar.com`
