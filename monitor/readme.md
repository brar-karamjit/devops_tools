# kube-prometheus-stack (monitor)

This folder contains Helm values for deploying [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) directly with Helm into the `monitor` namespace.

Grafana is exposed at `https://grafana.karamjitbrar.com`.

## Management model

Prometheus stack is managed with Helm (not an Argo CD `Application` in this folder).

- Chart: `prometheus-community/kube-prometheus-stack`
- Release name: `home-kube-prometheus`
- Namespace: `monitor`
- Values file: `argocd-apps/monitor/values.yaml`

Istio scrape monitors are managed separately via `argocd-apps/kiali/istio-metrics.yaml`.

## Files

| File | Purpose |
|------|---------|
| `values.yaml` | Helm values for Grafana ingress, dashboard, and Grafana settings |
| `readme.md` | Install and operations notes |

## Install / Upgrade

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install home-kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitor \
  --create-namespace \
  --version 65.5.0 \
  -f argocd-apps/monitor/values.yaml
```

## How to update settings

1. Edit `argocd-apps/monitor/values.yaml`
2. Re-run `helm upgrade --install ... -f argocd-apps/monitor/values.yaml`

Common sections to edit:
- **Dashboards** — `grafana.dashboards`
- **Ingress / hostname / TLS** — `grafana.ingress`
- **Grafana settings** — `grafana.grafana.ini`

## Upgrade chart version

1. Change `--version` in the Helm command
2. Check the [chart changelog](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/CHANGELOG.md) for breaking changes
3. Run Helm upgrade

## Verification

```bash
# Check running pods
kubectl get pods -n monitor

# Check Helm release
helm list -n monitor | grep home-kube-prometheus

# Browse Grafana
open https://grafana.karamjitbrar.com
```
