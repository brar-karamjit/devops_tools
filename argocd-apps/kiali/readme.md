# Kiali

Kiali is deployed from this folder via the ArgoCD ApplicationSet (`all-apps-generator`).

## How this is deployed with ArgoCD

- `argocd/applicationset-all.yaml` scans `argocd-apps/*`.
- Because `kiali` is **not excluded**, ArgoCD auto-creates an Application named `kiali-app`.
- ArgoCD syncs manifests from `argocd-apps/kiali/`.

This folder also contains Istio monitoring CRs (`PodMonitor` / `ServiceMonitor`) used by Prometheus:

- `istio-metrics.yaml` creates monitors in namespace `monitor` so the `home-kube-prometheus` release can scrape Istio proxy and istiod metrics.

## Prerequisites

- ArgoCD and ApplicationSet are running.
- NGINX ingress controller is installed (`ingressClassName: nginx`).
- cert-manager and `letsencrypt-prod` ClusterIssuer exist (for TLS cert).
- DNS for `kiali.karamjitbrar.com` points to your ingress LB.

## Required step for basic auth ingress

This manifest enables NGINX basic auth:

- `nginx.ingress.kubernetes.io/auth-type: basic`
- `nginx.ingress.kubernetes.io/auth-secret: kiali-basic-auth`

So the secret **must exist** in namespace `kiali` before/with ingress sync.

Create it with your chosen username/password:

```bash
kubectl -n kiali create secret generic kiali-basic-auth \
  --from-literal=auth="$(htpasswd -nbB admin '<YOUR_PASSWORD>')"
```

Why this is needed: NGINX expects the `auth` key in `htpasswd` format, not a plain password string.

## Install / Sync steps

1. Push changes in this repo (if any).
2. Ensure ApplicationSet exists:
   ```bash
   kubectl get applicationset -n argocd all-apps-generator
   ```
3. Ensure the generated app exists:
   ```bash
   kubectl get applications -n argocd | grep kiali-app
   ```
4. Create `kiali-basic-auth` secret (command above).
5. Sync from ArgoCD UI or CLI:
   ```bash
   argocd app sync kiali-app
   ```

## Verify

```bash
kubectl get pods -n kiali
kubectl get svc -n kiali
kubectl get ingress -n kiali
kubectl describe ingress kiali-ingress -n kiali
```

Open: https://kiali.karamjitbrar.com

## Troubleshooting 503

If you get `503` from ingress:

1. Check ingress events for annotation parse errors:
   ```bash
   kubectl describe ingress kiali-ingress -n kiali
   ```
2. If you see missing `kiali-basic-auth`, recreate secret in `kiali` namespace.
3. Confirm service backend exists and endpoints are ready:
   ```bash
   kubectl get svc,endpoints -n kiali
   ```
4. Confirm ingress controller can see this ingress:
   ```bash
   kubectl get ingress -A | grep kiali
   ```

## Files

| File | Purpose |
| --- | --- |
| `kiali.yaml` | RBAC, ConfigMap, Deployment, Service, Ingress |
| `istio-metrics.yaml` | Istio `PodMonitor` and `ServiceMonitor` for Prometheus scraping |
| `readme.md` | Install and ops notes |
