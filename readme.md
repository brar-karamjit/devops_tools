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
3. External Secrets Operator (ESO) + Bitwarden bootstrap
4. Istio
5. Monitoring (kube-prometheus-stack)
6. Argo CD + ApplicationSet (deploy `argocd-apps/*`)

This order avoids dependency issues:
- cert-manager HTTP01 solver needs ingress
- Istio metrics scraping needs Prometheus CRDs
- ESO Bitwarden provider needs cert-manager for its SDK server TLS cert
- Argo CD is last to take over GitOps-managed app deployments

## 1) Ingress Controller

If using k3s default Traefik, disable it first (as per your k3s setup), then install ingress-nginx.

If your Oracle Load Balancer backend set depends on specific node ports, pin them during install so they never change on upgrade/recreate.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.nodePorts.http=32319 \
  --set controller.service.nodePorts.https=30418
```

Verify ports:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
# Expected PORT(S): 80:32319/TCP,443:30418/TCP

kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}{"\n"}{.spec.ports[?(@.port==443)].nodePort}{"\n"}'
```

Notes:

- Keep `controller.service.type=LoadBalancer` so the service still exposes 80/443 and assigns an external IP.
- OCI backend sets should target the pinned node ports (for example `32319` and `30418`) on each Kubernetes node.
- If you ever need different ports, update both Helm values and OCI backend set ports together.

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

## 3) External Secrets Operator (ESO) + Bitwarden

See `external-secrets/readme.md` for full setup and the Bitwarden secret naming table.

Quick summary:
```bash
# Apply SDK server TLS cert (requires cert-manager above)
kubectl apply -f external-secrets/secret-stores/bitwarden-sdk-tls.yaml

# Bootstrap the Bitwarden machine account token (one-time, never commit real value)
kubectl -n external-secrets create secret generic bitwarden-access-token \
  --from-literal=token='<YOUR_TOKEN>'

# Install ESO with Bitwarden SDK server
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm upgrade --install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set bitwarden-sdk-server.enabled=true \
  -f external-secrets/values.yaml \
  --wait

# Fill in caBundle, organizationID, projectID then apply ClusterSecretStore
kubectl apply -f external-secrets/secret-stores/bitwarden-clustersecretstore.yaml
```

## 4) Istio

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

## 5) Monitoring

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

## 6) Argo CD + App of Apps

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
- `argocd-apps/` → GitOps-managed app manifests (each app folder contains ExternalSecret if it has secrets)
- `cert-manager/` → Helm values + ClusterIssuer
- `external-secrets/` → ESO Helm values + ClusterSecretStore + bootstrap runbook
- `istio/` → Istio setup notes
- `monitor/` → Prometheus Helm values + Istio scrape monitors

## Resume Project Write-Up

This repository is a **personal homelab project** that demonstrates production-like Kubernetes platform patterns.

### Project Title Ideas

- Kubernetes Homelab Platform Automation (GitOps)
- GitOps-Based Kubernetes Platform (Argo CD + Istio)
- Secure Kubernetes Homelab with External Secrets and Observability

### One-Line Resume Summary

Built a GitOps-driven Kubernetes homelab platform using Argo CD, cert-manager, External Secrets (Bitwarden), Istio, and Prometheus/Grafana to automate secure multi-service deployments and cluster operations.

### Role-Targeted Resume Bullets

#### DevOps Engineer

- Built an Argo CD `ApplicationSet` pattern that auto-discovers and deploys apps from `argocd-apps/*`, with automated sync, prune, self-heal, and namespace creation.
- Standardized cluster bootstrap using Helm and Kubernetes manifests for ingress, TLS, secret management, service mesh, monitoring, and GitOps app delivery.
- Integrated External Secrets Operator with Bitwarden-backed `ClusterSecretStore` and TLS-secured SDK server to avoid storing application secrets in Git.

#### Site Reliability Engineer (SRE)

- Implemented declarative, repeatable cluster provisioning in a homelab to reduce manual drift and improve recovery consistency after environment rebuilds.
- Enabled self-healing GitOps sync policies (`prune`, `selfHeal`, and server-side apply) to continuously reconcile workloads to desired state.
- Added observability and traffic visibility with Prometheus scraping for Istio components and Kiali integration for service graph and mesh troubleshooting.

#### Platform Engineer

- Designed reusable platform primitives across namespaces: ingress + DNS routing, automated TLS (`cert-manager`), shared secret store integration, and service mesh onboarding.
- Built an app-onboarding model where each service folder under `argocd-apps/` can be promoted through a consistent Kubernetes/GitOps structure.
- Operationalized cross-cutting platform services (Argo CD, ESO, Istio, monitoring) to provide a foundation for deploying both stateless and stateful workloads.

### Homelab-Safe Wording Guide

Use phrasing like:

- "personal homelab"
- "production-like patterns"
- "self-hosted Kubernetes environment"
- "designed and implemented"

Avoid phrasing like:

- "owned production infrastructure"
- "supported customer-facing SLA"
- "reduced company cloud costs"

### Interview Support

For stronger resume credibility, add supporting evidence in this repo:

- Argo CD screenshot showing app health/sync across generated applications
- Kiali service graph screenshot
- Grafana dashboard screenshot for Istio/control-plane metrics
- `kubectl get applications -n argocd` and `kubectl get externalsecret -A` output samples

For copy-paste versions (concise and detailed), use `resume-project.md`.
