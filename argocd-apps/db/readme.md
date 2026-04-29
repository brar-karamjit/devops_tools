# n8n DB (separate ArgoCD app)

This folder contains the database manifests for the `n8n` app.

Because `argocd/applicationset-all.yaml` scans `argocd-apps/*`, this folder is automatically deployed as a separate ArgoCD application named `db-app` in namespace `db`.

## Manifests
- `externalsecret.yaml`: syncs DB credentials from Bitwarden to `n8n-db-secrets`.
- `deployment.yaml`: creates a PostgreSQL StatefulSet and ClusterIP Service.

## Bitwarden secrets required
- `n8n-db-username`
- `n8n-db-password`

## In-cluster connection for n8n
- Host: `n8n-db.db.svc.cluster.local`
- Port: `5432`