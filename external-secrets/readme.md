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

> Install ESO + Bitwarden before Argo CD.
>
> Argo CD can apply `ExternalSecret` manifests, but those resources cannot produce real Kubernetes `Secret` objects until the ESO controller and the Bitwarden `ClusterSecretStore` already exist and are healthy.

### 1. Install `cert-manager`

Purpose: issues the TLS certificate used by the in-cluster Bitwarden SDK server.

### 2. Create the `external-secrets` namespace

Purpose: gives ESO and the Bitwarden SDK server a dedicated place to run.

```bash
kubectl create namespace external-secrets
```

### 3. Apply the Bitwarden SDK TLS certificate manifest

Purpose: creates the certificate resource that secures ESO's HTTPS connection to the Bitwarden SDK server.

```bash
kubectl apply -f external-secrets/secret-stores/bitwarden-sdk-tls.yaml
```

### 4. Create the Bitwarden machine account token secret

Purpose: stores the Bitwarden access token in Kubernetes so the SDK server can authenticate to Bitwarden Secrets Manager.

```bash
kubectl -n external-secrets create secret generic bitwarden-access-token \
  --from-literal=token='<YOUR_TOKEN>'
```

### 5. Install ESO with the Bitwarden SDK server enabled

Purpose: deploys the ESO controller that watches `ExternalSecret` resources and the Bitwarden SDK server that talks to Bitwarden.

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

### 6. Get the CA bundle from the generated TLS secret

Purpose: extracts the CA data ESO will trust when calling the Bitwarden SDK server over HTTPS.

```bash
kubectl -n external-secrets get secret bitwarden-tls-certs \
  -o jsonpath='{.data.ca\.crt}'
```

Paste the base64 output into `caBundle` in `bitwarden-clustersecretstore.yaml`.

### 7. Fill in and apply `bitwarden-clustersecretstore.yaml`

Purpose: creates the cluster-wide connection definition ESO uses to know where Bitwarden is and how to read your secrets.

Set these fields first:

- `caBundle`
- `organizationID`
- `projectID`

Then apply it:

```bash
kubectl apply -f external-secrets/secret-stores/bitwarden-clustersecretstore.yaml
```

### 8. Verify the `ClusterSecretStore` is ready

Purpose: confirms ESO can reach Bitwarden successfully before any app tries to consume secrets.

```bash
kubectl get clustersecretstore bitwarden-secretsmanager
```

### 9. Install Argo CD after ESO + Bitwarden is working

Purpose: once the secret backend is healthy, Argo CD can safely deploy app manifests that include `ExternalSecret` resources.

## Bitwarden Secret Naming Convention

| App   | Bitwarden Secret Name        | k8s Secret Key       |
|-------|------------------------------|----------------------|
| momo  | django-secret-key       | django-secret-key    |
| momo  | gemini-api-key          | gemini-api-key       |
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