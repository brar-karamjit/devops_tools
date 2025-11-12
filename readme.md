# DevOps Tools Repository

This repository stores DevOps manifests and helpers for deploying multiple applications to Kubernetes using ArgoCD (GitOps).

## Table of contents

- Repository layout
- Quickstart (install ArgoCD + deploy)
- How Application folders work
- Adding & updating apps
- Troubleshooting

## Repository layout

Top-level structure (important files and folders):

```
devops_tools/
├── argocd/               # ArgoCD controller install values & ApplicationSet
│   ├── applicationset-all.yaml
│   └── values.yaml
├── argocd-apps/          # Application manifests (one subfolder per app)
│   ├── invm/             # Inventory Manager (Django)
│   │   └── readme.md
│   └── momo/             # Moment In Motion (Django)
│       ├── deployment.yaml
│       └── readme.md
├── readme.md             # This file
```

Note: the layout above mirrors how the included `applicationset-all.yaml` discovers and generates ArgoCD Applications — each subfolder in `argocd-apps/` represents a single app.

## Quickstart

Prerequisites:

- A Kubernetes cluster
- kubectl configured to access the cluster
- Helm (recommended for installing ArgoCD)

1) Install ArgoCD (example using Helm):

```bash
# add argo helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# install with the provided values file
helm install argocd argo/argo-cd -f ./argocd/values.yaml -n argocd --create-namespace
```

2) Access the ArgoCD UI:

```bash
# get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# port-forward the server (optional)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Browse to https://localhost:8080 and sign in as `admin` with the password from the secret.

3) Deploy the ApplicationSet to generate apps from `argocd-apps/`:

```bash
kubectl apply -f argocd/applicationset-all.yaml
```

This ApplicationSet will:

- scan `argocd-apps/` for subfolders
- create an ArgoCD Application resource for each subfolder
- target the namespace matching the folder name by default (check the ApplicationSet template)

## How application folders should look

Each application folder under `argocd-apps/` should generally contain:

- Kubernetes manifests (Deployment, Service, ConfigMap, Ingress, etc.) or a kustomize/helm chart
- a `readme.md` describing the application and any special deployment notes
- (optional) a `namespace` manifest, or rely on the ApplicationSet to create a namespace

Keep each app self-contained so the ApplicationSet can treat each folder as an independent unit.

## Adding and updating apps

To add a new app:

1. Create `argocd-apps/<app-name>/` and add manifests and `readme.md`
2. Commit and push to the repo
3. The ApplicationSet will detect the new folder and create the ArgoCD Application automatically

To update an app, modify the manifests in the corresponding folder and commit; ArgoCD will sync the changes according to your sync policy.

## Troubleshooting

Check ArgoCD Application resources and status:

```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```

View pod logs in the target namespace:

```bash
kubectl logs -n <app-namespace> deployment/<deployment-name>
```

If auto-sync fails, you can manually sync via the ArgoCD CLI or UI:

```bash
argocd app sync <app-name>
```

## Contributing

1. Fork and branch
2. Add or update app folders under `argocd-apps/`
3. Test your changes and open a PR

## License

MIT