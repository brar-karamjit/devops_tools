# Istio setup (hello + momo)

This repo uses Istio sidecar injection for the `hello` and `momo` namespaces, while keeping the existing NGINX ingress (hostPort on 80/443) as the edge entry point. We intentionally do **not** add an Istio Gateway.

## What is applied by Argo CD

The ApplicationSet in `argocd/applicationset-all.yaml` tracks every folder under `argocd-apps/*`. That means any YAML inside `argocd-apps/hello` and `argocd-apps/momo` is part of the desired state and is applied automatically on sync.

We added:

- `argocd-apps/hello/istio.yaml`
- `argocd-apps/momo/istio.yaml`

These include a `PeerAuthentication` set to `PERMISSIVE` for each namespace. This keeps traffic working during the first mesh rollout and avoids breaking calls from non-meshed pods.

Note: If you do not want Argo CD to create/manage namespaces, do not include a `Namespace` manifest in these files. Instead, label the namespace manually (see below).

## Why this design

- **No Istio Gateway**: NGINX ingress already terminates TLS and receives traffic via `hostPort` on every node. Adding an Istio Gateway would duplicate edge responsibilities and complicate routing.
- **Gradual mTLS**: `PERMISSIVE` allows both mTLS and plain traffic, so existing services can talk while sidecars roll out. Once stable, you can move to `STRICT`.
- **Argo CD driven**: keeping mesh config in app folders makes the desired state explicit and auditable with the same GitOps flow as the apps themselves.

## Manual steps

1) Install Istio base and control plane (no gateway):

```bash
kubectl create namespace istio-system
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system
```

2) Ensure the `hello` and `momo` namespaces are labeled for injection:

```bash
kubectl label namespace hello istio-injection=enabled
kubectl label namespace momo istio-injection=enabled
```

3) Restart deployments to inject sidecars:

```bash
kubectl rollout restart deploy -n hello
kubectl rollout restart deploy -n momo
```

## Optional hardening

When ready, switch `PeerAuthentication` to `STRICT` per namespace. You may also add `DestinationRule` resources for service-level mTLS enforcement.

## Kiali UI

Kiali is deployed via `argocd-apps/kiali/kiali.yaml` and exposed through the existing NGINX ingress at `kiali.karamjitbrar.com`. We keep the ingress path simple (direct to the Kiali service) and avoid an Istio Gateway for the same reason as the apps: NGINX already handles edge traffic via hostPort.

Kiali authentication is set to `anonymous` in this setup. If you want authenticated access, switch to a supported strategy (for example, OpenID) and add the required secrets.

We also enable NGINX basic auth at the ingress layer. This is the easiest way to add login protection without changing Kiali's internal auth strategy. The ingress expects a secret named `kiali-basic-auth` in the `kiali` namespace.
