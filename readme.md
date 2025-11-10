# DevOps Tools Repository

This repository contains DevOps configurations and tools for managing Kubernetes applications using ArgoCD. It provides automated deployment pipelines for multiple applications through GitOps practices.

## Repository Structure

```
devops_tools/
├── argocd/
│   ├── applicationset-all.yaml    # ArgoCD ApplicationSet for auto-discovering apps
│   └── values.yaml                # ArgoCD Helm chart values
├── argocd-apps/
│   ├── invm/                      # Inventory Manager application manifests
│   │   └── readme.md
│   └── momo/                      # Moment in Motion application manifests
│       ├── deployment.yaml
│       └── readme.md
└── readme.md                      # This file
```

## Applications

### Inventory Manager (invm)
A Django-based inventory management system with barcode scanning capabilities.

### Moment in Motion (momo)
A motion tracking application built with Django.

## Prerequisites

- Kubernetes cluster
- ArgoCD installed in the cluster
- Helm (for ArgoCD installation)
- kubectl configured to access your cluster

## Installation and Setup

### 1. Install ArgoCD

```bash
# Add ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD with custom values
helm install argocd argo/argo-cd -f ./argocd/values.yaml -n argocd --create-namespace
```

### 2. Access ArgoCD UI

```bash
# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI (or use NodePort 30011 if configured)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Navigate to https://localhost:8080 and login with username `admin` and the password from above.

### 3. Deploy ApplicationSet

Apply the ApplicationSet to automatically create ArgoCD applications for each app in the `argocd-apps/` directory:

```bash
kubectl apply -f argocd/applicationset-all.yaml
```

This will:
- Scan the `argocd-apps/` directory for subfolders
- Create an ArgoCD Application for each folder found
- Deploy the applications to namespaces matching the folder names

## Adding New Applications

1. Create a new folder under `argocd-apps/` (e.g., `argocd-apps/myapp/`)
2. Add your Kubernetes manifests to the folder
3. Add a `readme.md` describing the application
4. Commit and push changes
5. ArgoCD will automatically detect and deploy the new application

## Application Manifest Requirements

Each application folder should contain:
- Kubernetes manifests (deployments, services, configmaps, etc.)
- A `readme.md` file describing the application
- Proper namespace declarations in manifests (will be created automatically)

## Updating Applications

Simply update the manifests in the respective `argocd-apps/<app>/` folder and commit. ArgoCD will automatically sync the changes due to the automated sync policy.

## Troubleshooting

### Check Application Status
```bash
kubectl get applications -n argocd
```

### View Application Logs
```bash
kubectl logs -n <app-namespace> deployment/<app-deployment>
```

### Manual Sync
If auto-sync fails, manually sync through ArgoCD UI or CLI:
```bash
argocd app sync <app-name>
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Add your application manifests
4. Test deployment
5. Submit a pull request

## License

This repository is licensed under the MIT License.