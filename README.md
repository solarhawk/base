# Dorkomen Chart

A curated Kubernetes platform deployment using GitOps (FluxCD) that includes:

- **Cert-Manager** - TLS certificate management
- **ECK Operator 3.x** - Elastic Cloud on Kubernetes (supports Elastic 9.x)
- **Elastic Stack 9.x** - Elasticsearch, Kibana, Fleet Server, Elastic Agent, APM Server
- **OpenTelemetry Operator** - Auto-instrumentation for distributed tracing
- **GitLab** - DevOps platform (CE edition)
- **GitLab Runner** - CI/CD runner
- **ArgoCD** - GitOps continuous delivery
- **n8n** - Workflow automation

---

## Quick Start

### Prerequisites

- **Kubernetes Cluster** (v1.28+) - Rancher Desktop, k3s, or compatible
- **Traefik Ingress Controller** (included with k3s/Rancher Desktop)
- **kubectl** configured to access your cluster
- **helm** (v3.x)

### Create Your Own Repository

**Important:** Create your own copy of this chart. You will configure your deployment in your repository - no secrets should ever be committed to git.

**Option A: Fork on GitHub**

1. Click "Fork" on GitHub to create your own copy
2. Clone your fork:
   ```bash
   git clone https://github.com/YOUR_USERNAME/YOUR_FORK.git
   cd YOUR_FORK
   ```

**Option B: Create a new repository**

```bash
# Clone this repository
git clone https://github.com/solarhawk/base.git dorkomen-chart
cd dorkomen-chart

# Remove the original remote and add your own
git remote remove origin
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Push to your repository
git push -u origin main
```

---

## Step 1: Install FluxCD

```bash
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Wait for controllers to be ready
kubectl -n flux-system rollout status deployment/helm-controller
kubectl -n flux-system rollout status deployment/source-controller
```

## Step 2: Install ECK CRDs

The Elastic Stack requires ECK Operator CRDs:

```bash
kubectl create -f https://download.elastic.co/downloads/eck/3.2.0/crds.yaml
```

## Step 3: Create Required Secrets

**These secrets must be created before deploying the chart. Never commit secrets to git.**

### Option A: Let's Encrypt with Cloudflare (Production)

```bash
# Create cert-manager namespace
kubectl create namespace cert-manager

# Create Cloudflare API token secret
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN
```

To create a Cloudflare API token:
1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com) → My Profile → API Tokens
2. Create token with permissions: `Zone:Zone:Read` and `Zone:DNS:Edit`
3. Zone Resources: Include → Specific zone → your domain

### Option B: Let's Encrypt with Route53 (Production)

```bash
kubectl create namespace cert-manager

kubectl create secret generic route53-credentials \
  --namespace cert-manager \
  --from-literal=access-key-id=YOUR_ACCESS_KEY \
  --from-literal=secret-access-key=YOUR_SECRET_KEY
```

### Option C: Self-Signed Certificates (Development)

No secrets required. Self-signed CA is created automatically.

## Step 4: Configure Your Deployment

Edit `chart/values.yaml` in your fork:

```yaml
# Set your domain
domain: yourdomain.com

# Choose certificate issuer
# Options: "dorkomen-ca" (self-signed), "letsencrypt-staging", "letsencrypt-prod"
clusterIssuer: letsencrypt-prod

# Update email for Let's Encrypt
certManager:
  clusterIssuers:
    letsencrypt:
      email: "your-email@example.com"
```

### Enable/Disable Components

```yaml
certManager:
  enabled: true

eckOperator:
  enabled: true

elasticStack:
  enabled: true

gitlab:
  enabled: true

gitlabRunner:
  enabled: true

argocd:
  enabled: true

n8n:
  enabled: true
```

**Commit and push your configuration changes:**

```bash
git add chart/values.yaml
git commit -m "Configure for my environment"
git push origin main
```

## Step 5: Create FluxCD Resources

### Create GitRepository

For public repositories:

```bash
kubectl apply -f - <<'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: dorkomen
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/YOUR_USERNAME/YOUR_FORK.git
  ref:
    branch: main
EOF
```

For private repositories:

```bash
# Create authentication secret
kubectl create secret generic dorkomen-git-auth \
  --namespace flux-system \
  --from-literal=username=YOUR_USERNAME \
  --from-literal=password=YOUR_GITHUB_TOKEN

# Create GitRepository with secret reference
kubectl apply -f - <<'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: dorkomen
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/YOUR_USERNAME/YOUR_FORK.git
  ref:
    branch: main
  secretRef:
    name: dorkomen-git-auth
EOF
```

### Create HelmRelease

```bash
kubectl apply -f - <<'EOF'
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: dorkomen
  namespace: flux-system
spec:
  interval: 10m
  timeout: 15m
  chart:
    spec:
      chart: ./chart
      sourceRef:
        kind: GitRepository
        name: dorkomen
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
EOF
```

## Step 6: Monitor Deployment

```bash
# Watch HelmReleases reconcile
kubectl get helmreleases -A -w

# Check all pods
kubectl get pods -A

# Force immediate reconciliation
kubectl annotate gitrepository dorkomen -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

### Expected Deployment Order

1. cert-manager → Ready
2. eck-operator → Ready
3. elastic-stack, opentelemetry-operator, argocd, n8n → Ready (parallel)
4. gitlab → Ready (takes 5-10 minutes)
5. gitlab-runner → Ready

## Step 7: Configure DNS

### Production (Real Domain)

Add DNS A records pointing to your cluster's ingress IP:

- `argocd.yourdomain.com`
- `gitlab.yourdomain.com`
- `registry.yourdomain.com`
- `n8n.yourdomain.com`
- `kibana.yourdomain.com`

### Local Development

Add to your hosts file:

**Windows:** `C:\Windows\System32\drivers\etc\hosts`
**Linux/Mac:** `/etc/hosts`

```
127.0.0.1 argocd.yourdomain.local
127.0.0.1 gitlab.yourdomain.local
127.0.0.1 n8n.yourdomain.local
127.0.0.1 kibana.yourdomain.local
```

## Step 8: Access Services

| Service | URL | Credentials |
|---------|-----|-------------|
| ArgoCD | https://argocd.yourdomain.com | `admin` / see below |
| GitLab | https://gitlab.yourdomain.com | `root` / see below |
| Kibana | https://kibana.yourdomain.com | `elastic` / see below |
| n8n | https://n8n.yourdomain.com | Create on first login |

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

---

## Secrets Reference

All secrets are created in-cluster using `kubectl`. **Never commit secrets to git.**

| Secret | Namespace | Purpose | When Required |
|--------|-----------|---------|---------------|
| `cloudflare-api-token` | cert-manager | Cloudflare DNS-01 challenge | Let's Encrypt with Cloudflare |
| `route53-credentials` | cert-manager | Route53 DNS-01 challenge | Let's Encrypt with Route53 |
| `dorkomen-git-auth` | flux-system | Private git repository access | Private repos only |

### Secrets Created Automatically

These secrets are created by the chart and do not require manual setup:

| Secret | Namespace | Purpose |
|--------|-----------|---------|
| `gitlab-gitlab-runner-secret` | gitlab | GitLab Runner registration token |
| `elasticsearch-es-elastic-user` | elastic-stack | Elasticsearch admin password |
| `argocd-argocd-initial-admin-secret` | argocd | ArgoCD admin password |
| `gitlab-gitlab-initial-root-password` | gitlab | GitLab root password |

---

## Updating Your Deployment

With GitOps, updates are automatic:

```bash
# Make changes to values.yaml
vim chart/values.yaml

# Commit and push
git add . && git commit -m "Update configuration" && git push

# Flux auto-syncs within 5 minutes, or force immediately:
kubectl annotate gitrepository dorkomen -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

---

## Resource Requirements

Minimum recommended resources (all components enabled):

| Component | CPU Request | Memory Request |
|-----------|-------------|----------------|
| cert-manager | 50m | 64Mi |
| eck-operator | 100m | 150Mi |
| elasticsearch | 500m | 1Gi |
| kibana | 250m | 512Mi |
| apm-server | 100m | 256Mi |
| fleet-server | 100m | 256Mi |
| elastic-agent | 200m | 1Gi |
| opentelemetry-operator | 100m | 128Mi |
| gitlab | 1000m | 4Gi |
| argocd | 250m | 256Mi |
| n8n | 100m | 256Mi |

**Total:** ~3 CPU cores, ~8GB RAM minimum

---

## Troubleshooting

### Check HelmRelease Status

```bash
kubectl get helmreleases -A
kubectl describe helmrelease <name> -n flux-system
```

### Check Flux Logs

```bash
kubectl logs -n flux-system deployment/helm-controller
```

### Check ClusterIssuers

```bash
kubectl get clusterissuers
kubectl describe clusterissuer letsencrypt-prod
```

### Check Certificates

```bash
kubectl get certificates -A
kubectl describe certificate <name> -n <namespace>
```

### Force Reconciliation

```bash
# Reconcile git source
kubectl annotate gitrepository dorkomen -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite

# Reconcile specific HelmRelease
kubectl annotate helmrelease <name> -n flux-system \
  reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite
```

---

## Uninstalling

```bash
# Remove HelmRelease and GitRepository
kubectl delete helmrelease dorkomen -n flux-system
kubectl delete gitrepository dorkomen -n flux-system

# Remove FluxCD (optional)
kubectl delete -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml

# Remove ECK CRDs (optional)
kubectl delete -f https://download.elastic.co/downloads/eck/3.2.0/crds.yaml

# Clean up namespaces
kubectl delete namespace cert-manager eck-operator gitlab gitlab-runner \
  argocd n8n elastic-stack opentelemetry-operator-system
```

---

## Architecture

### Namespace Layout

```
flux-system                    - FluxCD controllers + HelmReleases
cert-manager                   - Certificate management
eck-operator                   - ECK Operator 3.x
elastic-stack                  - Elasticsearch, Kibana, Fleet, Agent, APM
opentelemetry-operator-system  - OpenTelemetry Operator
gitlab                         - GitLab CE + PostgreSQL + Redis + MinIO
gitlab-runner                  - GitLab Runner
argocd                         - ArgoCD
n8n                            - n8n workflow automation
```

### Component Dependencies

```
FluxCD + ECK CRDs (prerequisites)
    │
    └── cert-manager
            │
            ├── eck-operator
            │       │
            │       └── elastic-stack
            │
            ├── opentelemetry-operator
            │
            ├── argocd
            │
            ├── n8n
            │
            └── gitlab
                    │
                    └── gitlab-runner
```

---

## License

See [LICENSE](LICENSE) for details.
