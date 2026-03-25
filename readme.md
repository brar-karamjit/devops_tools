# DevOps Tools

Operational runbook for building this cluster from a base k3s setup.

## Cluster Setup

### Prerequisites

- 2 VMs: one control-plane and one worker
- k3s installed and nodes joined
- External load balancer configured and pointing to the cluster ingress entry
- `kubectl` and `helm` installed on your admin machine
- DNS records created for app domains (for example Grafana and Kiali)

### Verify baseline

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

You should see both nodes in `Ready` state before continuing.

## Install Order (from this repo baseline)

1. Ingress controller
2. cert-manager
3. Istio
4. Monitoring (kube-prometheus-stack)
5. Argo CD + ApplicationSet (deploy `argocd-apps/*`)

This order avoids dependency issues:
- cert-manager HTTP01 solver needs ingress
- Istio metrics scraping needs Prometheus CRDs
- Argo CD is last to take over GitOps-managed app deployments

## 1) Ingress Controller

If using k3s default Traefik, disable it first (as per your k3s setup), then install ingress-nginx:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

## 2) cert-manager

Install cert-manager (Helm values in `cert-manager/values.yaml`):

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

Apply ClusterIssuer:

```bash
kubectl apply -f cert-manager/cluster-issuer.yaml
```

## 3) Istio

Install Istio base + control plane:

```bash
kubectl create namespace istio-system
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm upgrade --install istio-base istio/base -n istio-system
helm upgrade --install istiod istio/istiod -n istio-system
```

Enable sidecar injection for app namespaces and restart workloads:

```bash
kubectl label namespace hello istio-injection=enabled --overwrite
kubectl label namespace momo istio-injection=enabled --overwrite
kubectl rollout restart deploy -n hello
kubectl rollout restart deploy -n momo
```

## 4) Monitoring

Install kube-prometheus-stack (values in `monitor/values.yaml`):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install home-kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitor \
  --create-namespace \
  --version 65.5.0 \
  -f monitor/values.yaml
```

Apply Istio scrape monitors (requires Prometheus Operator CRDs from previous step):

```bash
kubectl apply -f monitor/istio-metrics.yaml
```

## 5) Argo CD + App of Apps

Install Argo CD:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd \
  -f argocd/values.yaml \
  -n argocd \
  --create-namespace
```

Create the ApplicationSet (auto-deploys everything under `argocd-apps/*`):

```bash
kubectl apply -f argocd/applicationset-all.yaml
```

## Quick Verification

```bash
kubectl get pods -n ingress-nginx
kubectl get pods -n cert-manager
kubectl get pods -n istio-system
kubectl get pods -n monitor
kubectl get applications -n argocd
```

## Repo Layout (current)

- `argocd/` → Argo CD Helm values + ApplicationSet
- `argocd-apps/` → GitOps-managed app manifests
- `cert-manager/` → Helm values + ClusterIssuer
- `istio/` → Istio setup notes
- `monitor/` → Prometheus Helm values + Istio scrape monitors
