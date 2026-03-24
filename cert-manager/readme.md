# cert-manager

This folder now follows a standard Helm-based setup:

1. Install/upgrade cert-manager from the upstream Helm chart.
2. Apply the `ClusterIssuer` manifest separately.

## Files

| File | Purpose |
| --- | --- |
| `values.yaml` | Helm values for upstream `jetstack/cert-manager` chart |
| `cluster-issuer.yaml` | Let's Encrypt production `ClusterIssuer` |
| `readme.md` | Install and operations notes |

## Install / Upgrade cert-manager (Helm)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.1 \
  -f cert-manager/values.yaml \
  --wait
```

## Apply ClusterIssuer

Apply after cert-manager is up:

```bash
kubectl apply -f cert-manager/cluster-issuer.yaml
```

## Verification

```bash
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager.io
kubectl get clusterissuer letsencrypt-prod
```

## Notes

- `ClusterIssuer` is intentionally separate from Helm values; upstream chart values configure cert-manager components, not environment-specific issuer resources.
- If you need staging certs too, add another manifest (for example `cluster-issuer-staging.yaml`) and apply it the same way.
