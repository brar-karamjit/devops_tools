# momo DB (separate ArgoCD app)

This folder contains the database manifests for the `momo` app.

Because `argocd/applicationset-all.yaml` scans `argocd-apps/*`, this folder is automatically deployed as a separate ArgoCD application named `db-app` in namespace `db`.

## Manifests
- `externalsecret.yaml`: syncs DB credentials from Bitwarden to `momo-db-secrets`.
- `deployment.yaml`: creates a PostgreSQL StatefulSet and ClusterIP Service.

## Bitwarden secrets required
- `momo-db-name`
- `momo-db-username`
- `momo-db-password`

## In-cluster connection for momo
- Host: `momo-db.db.svc.cluster.local`
- Port: `5432`