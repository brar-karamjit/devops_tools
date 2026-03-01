# cert-manager

Installs [cert-manager](https://cert-manager.io/) and a Let's Encrypt `ClusterIssuer` via a single ArgoCD **multi-source** Application.

## Why is this excluded from the ApplicationSet?

The ApplicationSet (`all-apps-generator`) assumes every folder under `argocd-apps/` contains plain Kubernetes manifests to apply directly. cert-manager is different — it needs to:

1. **Install a Helm chart** from an external registry (`charts.jetstack.io`), not apply local manifests.
2. **Apply additional manifests** (the `ClusterIssuer`) that depend on CRDs created by the Helm chart.

If the ApplicationSet managed this folder, it would try to apply `cert-manager.yaml` as a resource — creating a nested Application (app-of-apps), which results in **two** ArgoCD Applications (`cert-manager-app` from the ApplicationSet and `cert-manager` from the inner manifest). The inner app is unaware of the ApplicationSet lifecycle, leading to sync conflicts and stuck deletions.

By excluding `cert-manager` from the generator and applying the Application directly, we get a clean single app with multi-source support.

## Directory structure

```
cert-manager/
├── cert-manager.yaml          # ArgoCD Application (multi-source)
├── readme.md
└── manifests/
    └── cluster-issuer.yaml    # ClusterIssuer for Let's Encrypt
```

## Installation

```bash
kubectl apply -f argocd-apps/cert-manager/cert-manager.yaml
```

ArgoCD will:
- Install the cert-manager Helm chart (with CRDs) into the `cert-manager` namespace.
- Apply everything in `manifests/` (the `ClusterIssuer`) from the Git repo.
- Auto-sync and self-heal going forward.

## Removal

```bash
kubectl delete application cert-manager -n argocd
```

This triggers the `resources-finalizer` which cascade-deletes all managed resources (cert-manager pods, CRDs, ClusterIssuer, etc.).

## Adding more manifests

Drop any additional YAML files into the `manifests/` directory. ArgoCD will pick them up on the next sync — no changes to `cert-manager.yaml` needed.
