# External Secrets Operator (ESO) + Bitwarden Secrets Manager

Manages all app secrets externally. Secrets live in Bitwarden Secrets Manager and are
pulled into Kubernetes at runtime by ESO. Git only contains references, never raw values.
Secrets survive namespace or full cluster deletion.

## Architecture

```
Argo CD applies ExternalSecret manifests
      ↓
ESO controller reads ExternalSecret
      ↓
ESO calls bitwarden-sdk-server (in-cluster, HTTPS)
      ↓
bitwarden-sdk-server calls Bitwarden Secrets Manager API
      ↓
ESO creates / updates Kubernetes Secret in app namespace
      ↓
App pod reads Secret via secretKeyRef (unchanged)
```

## Prerequisites

- cert-manager installed (needed for bitwarden-sdk-server TLS cert)
- Bitwarden Secrets Manager account with an Organisation and Project
- Machine Account created, scoped to your Project, with an access token

## Install Order

1. cert-manager (already in this repo)
2. Apply bitwarden SDK TLS cert:
   ```bash
   kubectl apply -f external-secrets/secret-stores/bitwarden-sdk-tls.yaml
   ```
3. Bootstrap access token secret (one-time, never commit real value):
   ```bash
   kubectl -n external-secrets create secret generic bitwarden-access-token \
     --from-literal=token='<YOUR_MACHINE_ACCOUNT_TOKEN>'
   ```
4. Install ESO with Bitwarden SDK server enabled:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm repo update
   helm upgrade --install external-secrets external-secrets/external-secrets \
     -n external-secrets \
     --create-namespace \
     --set bitwarden-sdk-server.enabled=true \
     -f external-secrets/values.yaml \
     --wait
   ```
5. Get the CA bundle from the generated TLS cert:
   ```bash
   kubectl -n external-secrets get secret bitwarden-sdk-server-tls \
     -o jsonpath='{.data.ca\.crt}'
   ```
   Paste the base64 output into `caBundle` in `bitwarden-clustersecretstore.yaml`.

6. Fill in your org/project IDs in `bitwarden-clustersecretstore.yaml` then apply:
   ```bash
   kubectl apply -f external-secrets/secret-stores/bitwarden-clustersecretstore.yaml
   ```
7. Verify store is ready:
   ```bash
   kubectl get clustersecretstore bitwarden-secretsmanager
   ```

## Bitwarden Secret Naming Convention

| App   | Bitwarden Secret Name        | k8s Secret Key       |
|-------|------------------------------|----------------------|
| momo  | momo-django-secret-key       | django-secret-key    |
| momo  | momo-gemini-api-key          | gemini-api-key       |
| kiali | kiali-basic-auth-htpasswd    | auth                 |

## Verification After Cluster Rebuild

```bash
# Check ESO controller is healthy
kubectl get pods -n external-secrets

# Check ClusterSecretStore is Ready
kubectl get clustersecretstore

# Check ExternalSecrets are synced
kubectl get externalsecret -n momo
kubectl get externalsecret -n kiali

# Confirm k8s secrets were created
kubectl -n momo get secret momo-secrets
kubectl -n kiali get secret kiali-basic-auth

# Restart momo pod to pick up fresh secrets
kubectl -n momo rollout restart deployment momo-deployment
```

## Rotation

To rotate a secret:
1. Update the value in Bitwarden Secrets Manager.
2. Wait up to 1h for ESO to refresh (or delete and re-create the ExternalSecret to force immediate sync).
3. Restart the affected pod to inject the new env value.

## Disaster Recovery Test

```bash
# Simulate namespace deletion
kubectl delete namespace momo

# Argo CD auto-sync recreates namespace + ExternalSecret
# ESO sees the new ExternalSecret and fetches momo-secrets from Bitwarden
# Verify
kubectl -n momo get secret momo-secrets
kubectl -n momo get pods
```
