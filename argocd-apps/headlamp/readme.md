# Headlamp

Headlamp provides a lightweight Kubernetes UI deployed via the ApplicationSet. The manifests in this folder are applied into the `headlamp` namespace.

## What gets deployed

- ServiceAccount and ClusterRole/ClusterRoleBinding (read-only)
- Deployment and Service
- Ingress at `https://headlamp.karamjitbrar.com`

## Files

| File | Purpose |
| --- | --- |
| `headlamp.yaml` | All Headlamp resources (RBAC, Deployment, Service, Ingress) |

## Access

```bash
kubectl get pods -n headlamp
kubectl get ingress -n headlamp
```

Open: https://headlamp.karamjitbrar.com