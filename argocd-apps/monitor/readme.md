# kube-prometheus-stack (monitor)

This folder contains the [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) deployment managed as a standalone Argo CD `Application`. It powers the public Grafana instance at `https://grafana.karamjitbrar.com` with Prometheus, Alertmanager, and Grafana — all deployed into the `monitor` namespace.

## How it works

Unlike other apps in `argocd-apps/` (e.g. `hello`, `home`, `momo`), the monitor stack is **not** managed by the `all-apps-generator` ApplicationSet. It is explicitly excluded:

```yaml
# argocd/applicationset-all.yaml
directories:
  - path: 'argocd-apps/*'
  - path: 'argocd-apps/monitor'
    exclude: true
```

### Why the exclusion?

The ApplicationSet treats each folder in `argocd-apps/` as plain Kubernetes manifests and creates an ArgoCD Application to apply them. But `kube-prometheus.stack.yaml` is itself an ArgoCD `Application` resource that deploys a **Helm chart** (not plain manifests). Without the exclusion, ArgoCD would create two apps:

1. `monitor-app` — the ApplicationSet wrapper that applies the YAML file
2. `kube-prometheus-stack` — the actual Application defined inside the YAML

This is redundant and causes sync issues, so `monitor` is excluded from the ApplicationSet.

### How ArgoCD sees it

The `kube-prometheus-stack` Application is applied directly to the cluster:

```bash
kubectl apply -f argocd-apps/monitor/kube-prometheus.stack.yaml
```

ArgoCD's controller watches all `Application` CRs in the `argocd` namespace regardless of how they were created (ApplicationSet, Helm, CI pipeline, or `kubectl apply`). Once the CR exists, ArgoCD picks it up and manages syncing.

## Files

| File | Purpose |
|------|---------|
| `kube-prometheus.stack.yaml` | ArgoCD `Application` — pulls the upstream Helm chart (pinned to `65.5.0`), sets the release name, and inlines all Grafana/Prometheus configuration |

## How to update

### Change Grafana or Prometheus settings

1. Edit the inline `helm.values` block inside `kube-prometheus.stack.yaml`
2. Apply the change:
   ```bash
   kubectl apply -f argocd-apps/monitor/kube-prometheus.stack.yaml
   ```
3. ArgoCD will detect the diff and sync automatically

Common sections to edit:
- **Dashboards** — `grafana.dashboards` (inline JSON or ConfigMap references)
- **Ingress / hostname / TLS** — `grafana.ingress`
- **Grafana settings** — `grafana.grafana.ini` (maps to `grafana.ini` sections like `auth.anonymous`, `server`, `security`)
- **Alerting rules** — `additionalPrometheusRulesMap`
- **Scrape configs** — `prometheus.prometheusSpec.additionalScrapeConfigs`

### Upgrade the chart version

1. Change `targetRevision` in `kube-prometheus.stack.yaml` (currently `65.5.0`)
2. Check the [chart changelog](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/CHANGELOG.md) for breaking changes
3. Apply:
   ```bash
   kubectl apply -f argocd-apps/monitor/kube-prometheus.stack.yaml
   ```

### Force a sync via ArgoCD CLI

```bash
argocd app sync kube-prometheus-stack
```

## Verification

```bash
# Check the ArgoCD Application status
kubectl get application kube-prometheus-stack -n argocd

# Check running pods
kubectl get pods -n monitor

# Browse Grafana
open https://grafana.karamjitbrar.com
```
