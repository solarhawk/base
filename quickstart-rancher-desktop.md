# Dorkomen Chart Quick Start (Rancher Desktop)

Commands to deploy the Dorkomen chart on a fresh Rancher Desktop cluster.

> **Important**: The GitRepository and Kustomization resources must be named `dorkomen` exactly, as the HelmRelease references this name in its sourceRef.

## Prerequisites

- Rancher Desktop with Kubernetes enabled
- kubectl configured

## Step 1: Install FluxCD

```bash
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Wait for controllers
kubectl -n flux-system rollout status deployment/helm-controller
kubectl -n flux-system rollout status deployment/source-controller
```

## Step 2: Install ECK CRDs

```bash
kubectl create -f https://download.elastic.co/downloads/eck/3.2.0/crds.yaml
```

## Step 3: Create GitRepository

```bash
kubectl apply -f - <<'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: dorkomen
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/solarhawk/base.git
  ref:
    branch: dev/kustomize-overlays
EOF
```

## Step 4: Create Kustomization (Rancher Desktop)

```bash
kubectl apply -f - <<'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: dorkomen
  namespace: flux-system
spec:
  interval: 10m
  path: ./overlays/rancher-desktop
  prune: true
  sourceRef:
    kind: GitRepository
    name: dorkomen
EOF
```

## Step 5: Monitor Deployment

```bash
# Watch HelmReleases
kubectl get helmrelease -A -w

# Check all pods
kubectl get pods -A

# Force reconciliation if needed
kubectl annotate gitrepository dorkomen -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

## Step 6: Add Hosts Entries

Add to `C:\Windows\System32\drivers\etc\hosts` (run Notepad as Administrator):

```
127.0.0.1 argocd.dev.yourdomain.local
127.0.0.1 gitlab.dev.yourdomain.local
127.0.0.1 registry.dev.yourdomain.local
127.0.0.1 minio.dev.yourdomain.local
127.0.0.1 kas.dev.yourdomain.local
127.0.0.1 n8n.dev.yourdomain.local
127.0.0.1 kibana.dev.yourdomain.local
```

## Step 7: Access Services

| Service | URL |
|---------|-----|
| ArgoCD | https://argocd.dev.yourdomain.local |
| GitLab | https://gitlab.dev.yourdomain.local |
| Kibana | https://kibana.dev.yourdomain.local |
| n8n | https://n8n.dev.yourdomain.local |

### Get Credentials

```bash
# ArgoCD admin password
kubectl -n argocd get secret argocd-argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# GitLab root password
kubectl -n gitlab get secret gitlab-gitlab-initial-root-password \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Elasticsearch password
kubectl -n elastic-stack get secret elasticsearch-es-elastic-user \
  -o jsonpath="{.data.elastic}" | base64 -d && echo
```

## Troubleshooting

### Dorkomen HelmRelease Fails on First Install

The `dorkomen` HelmRelease may fail on initial deployment with an error about `Certificate` resources not found. This occurs because cert-manager CRDs are not yet available when the chart tries to create Certificate resources.

**Solution**: Wait for cert-manager to fully deploy, then force a retry:

```bash
kubectl annotate helmrelease dorkomen -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

### Force HelmRelease Retry

If a HelmRelease fails, force a retry:

```bash
kubectl annotate helmrelease <name> -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

### Check HelmRelease Status

```bash
kubectl get helmrelease -A
kubectl describe helmrelease <name> -n flux-system
```

### Check Flux Logs

```bash
kubectl logs -n flux-system deployment/helm-controller --tail=50
kubectl logs -n flux-system deployment/kustomize-controller --tail=50
```

### GitLab Runner Registration

The GitLab Runner pod may show registration errors until GitLab is fully operational. The runner will automatically retry registration. If issues persist, check the runner logs:

```bash
kubectl logs -n gitlab-runner -l app=gitlab-runner --tail=50
```
