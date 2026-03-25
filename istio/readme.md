# Istio setup (hello + momo)

Istio is used for sidecar injection in `hello` and `momo`.
NGINX ingress remains the edge entry point, so no Istio Gateway is used.

## GitOps-managed files

- `argocd-apps/hello/istio.yaml`
- `argocd-apps/momo/istio.yaml`

Both define namespace `PeerAuthentication` in `PERMISSIVE` mode for safe rollout.

## Quick start

```bash
# 1) Install Istio control plane
kubectl create namespace istio-system
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system

# 2) Enable sidecar injection on app namespaces
kubectl label namespace hello istio-injection=enabled --overwrite
kubectl label namespace momo istio-injection=enabled --overwrite

# 3) Restart workloads to get sidecars
kubectl rollout restart deploy -n hello
kubectl rollout restart deploy -n momo
```

## Notes

- Move `PeerAuthentication` from `PERMISSIVE` to `STRICT` only after validation.
- Kiali is exposed at `https://kiali.karamjitbrar.com` via NGINX ingress.
- Ingress basic auth uses secret `kiali-basic-auth` in namespace `kiali`.
