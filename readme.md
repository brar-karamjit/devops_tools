
# DevOps Tools

This repo contains manifests and helpers for deploying multiple apps to Kubernetes using ArgoCD (GitOps).

## Structure

```
devops_tools/
├── argocd/         # ArgoCD install values & ApplicationSet
│   ├── applicationset-all.yaml
│   └── values.yaml
├── argocd-apps/    # App manifests (one folder per app)
│   ├── invm/
│   └── momo/
├── readme.md
```

Each subfolder in `argocd-apps/` is a separate app. The ApplicationSet auto-discovers these and creates ArgoCD Applications.

## Quickstart

**Prerequisites:** Kubernetes cluster, `kubectl`, Helm.

1. **Install ArgoCD:**
	```bash
	helm repo add argo https://argoproj.github.io/argo-helm
	helm repo update
	helm install argocd argo/argo-cd -f ./argocd/values.yaml -n argocd --create-namespace
	```
2. **Get admin password & access UI:**
	```bash
	kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
	kubectl port-forward svc/argocd-server -n argocd 8080:443
	```
	Visit https://localhost:8080 (user: `admin`).
3. **Deploy apps:**
	```bash
	kubectl apply -f argocd/applicationset-all.yaml
	```

## App Folders

Each folder in `argocd-apps/` should have:
- Kubernetes manifests (Deployment, Service, etc.)
- `readme.md` for app info

## Add or Update Apps

**Add:**
1. Create `argocd-apps/<app-name>/` with manifests and `readme.md`
2. Commit & push
3. ApplicationSet will auto-create the ArgoCD Application

**Update:**
Modify manifests and commit. ArgoCD will sync changes.

## Troubleshooting

Check ArgoCD Applications:
```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```
View pod logs:
```bash
kubectl logs -n <app-namespace> deployment/<deployment-name>
```
Manual sync:
```bash
argocd app sync <app-name>
```

## Contributing

Fork, branch, add/update app folders, test, and open a PR.

## License

MIT