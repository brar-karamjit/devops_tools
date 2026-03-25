# kube-prometheus-stack (monitor)

Helm-managed Prometheus stack for namespace `monitor`.

- Values: `monitor/values.yaml`
- Istio scrape config: `monitor/istio-metrics.yaml`
- Grafana: `https://grafana.karamjitbrar.com`

## Quick start

```bash
# 1) Install/upgrade Prometheus stack (creates PodMonitor/ServiceMonitor CRDs)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install home-kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitor \
  --create-namespace \
  --version 65.5.0 \
  -f monitor/values.yaml

# 2) Apply Istio scrape monitors
kubectl apply -f monitor/istio-metrics.yaml
```

## Order

1. Install Istio.
2. Install/upgrade kube-prometheus-stack.
3. Apply `monitor/istio-metrics.yaml`.

## Update

- Edit `monitor/values.yaml`.
- Re-run the Helm command above.

## Verify

```bash
kubectl get pods -n monitor
helm list -n monitor | grep home-kube-prometheus
open https://grafana.karamjitbrar.com
```
