# Home Grafana Instance

This folder deploys the public-facing Grafana instance that powers `https://k8s.karamjitbrar.com`. Anonymous visitors now land on a lightweight Kubernetes pod overview (total pods plus cluster CPU/memory usage) so they immediately see live capacity data without logging in.

## Components

- `deployment.yaml` – Grafana deployment configured for anonymous read-only access and a custom default home dashboard.
- `service.yaml` – Exposes Grafana on port 80 inside the cluster.
- `ingress.yaml` – Publishes Grafana through the shared NGINX ingress controller.
- `dashboard-config.yaml` – Two ConfigMaps:
  - `home-grafana-dashboard` holds the JSON for the landing dashboard.
  - `home-grafana-provisioning` wires the dashboard into Grafana's provisioning system.

## Updating the landing dashboard

1. Edit the `home-dashboard.json` payload inside `dashboard-config.yaml`. The shipped dashboard relies on Prometheus metrics (`kube-state-metrics` + cAdvisor) using the Prometheus data source named `Prometheus`.
2. Keep the file mounted path the same (`/var/lib/grafana/dashboards/home/home-dashboard.json`) so the `GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH` environment variable continues to work.
3. Commit your changes and let Argo CD sync the application, or run `argocd app sync home` for an immediate rollout.
4. If you change PromQL queries, verify the required metrics exist in your cluster.

## Changing the default dashboard

- Update `GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH` in `deployment.yaml` if you rename the JSON file.
- Adjust the provisioning file (`home-dashboards.yaml`) if you need different folders or multiple dashboards.

## Verification checklist

After syncing:

- `kubectl get pods -n home` shows the Grafana pod restarted once (to pick up the new ConfigMaps).
- Opening `https://k8s.karamjitbrar.com/` in an incognito window displays the pod overview dashboard without requiring authentication.
