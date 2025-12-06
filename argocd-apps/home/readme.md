# Home kube-prometheus-stack

This folder now installs the full [`kube-prometheus-stack`](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) via Argo CD to power the public Grafana instance at `https://k8s.karamjitbrar.com`. Grafana is published through the shared NGINX ingress controller, enforces HTTPS, sets `GF_SERVER_ROOT_URL` appropriately, and enables anonymous read-only access for quick cluster visibility.

## Components

- `kube-prometheus-stack.yaml` – Argo CD `Application` that pulls the upstream Helm chart, pins the version, and injects the Grafana ingress and anonymous-auth configuration.

## Tweaking Grafana or Prometheus

- Update the inline `helm.values` block inside `kube-prometheus-stack.yaml` for any Grafana/Prometheus overrides (extra dashboards, datasources, alerting rules, etc.).
- To change the public hostname or TLS secret, edit the `grafana.ingress` section in the same file.
- Additional Grafana settings can be placed under `grafana.grafana.ini`. These keys map directly to sections in `grafana.ini` (for example `auth.anonymous`, `server`, `security`).

## Sync & Verification

1. Commit your changes and allow the `home-app` ApplicationSet entry to sync, or trigger manually with `argocd app sync home-app`.
2. Confirm `kubectl get pods -n home` shows the Prometheus, Alertmanager, and Grafana workloads from the Helm release.
3. Browse to `https://k8s.karamjitbrar.com` in an incognito window to ensure the Grafana welcome/dashboard page loads without authentication.
