# Multi-Cluster Setup Example

This directory contains configuration files for running Kargo with separate clusters for dev, staging, and prod environments, each with its own Argo CD instance.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ Dev Cluster (Control Plane)                                 │
│                                                              │
│  ┌────────────┐         ┌──────────────────┐               │
│  │   Kargo    │────────>│ Argo CD (dev)    │               │
│  │            │         │ guestbook-dev    │               │
│  └────────────┘         └──────────────────┘               │
│         │                                                    │
│         │ Manages promotions                                │
│         │                                                    │
└─────────┼────────────────────────────────────────────────────┘
          │
          ├─── (remote connection) ──>  Staging Cluster
          │                             ┌──────────────────┐
          │                             │ Argo CD (stg)    │
          │                             │ guestbook-staging│
          │                             └──────────────────┘
          │
          └─── (remote connection) ──>  Prod Cluster
                                        ┌──────────────────┐
                                        │ Argo CD (prod)   │
                                        │ guestbook-prod   │
                                        └──────────────────┘
```

## Setup Steps

### 1. Install Kargo (in dev cluster or separate management cluster)

```bash
# Switch to the cluster where you want Kargo
kubectl config use-context dev-cluster

# Install Kargo
helm repo add kargo https://charts.kargo.io
helm install kargo kargo/kargo \
  --namespace kargo \
  --create-namespace
```

### 2. Create Kargo Project

```bash
# Apply from parent directory
kubectl apply -f ../kargo/project.yaml
```

### 3. Configure Argo CD Credentials

**Option A: From YAML file (after editing with real passwords)**

```bash
# Edit argocd-credentials.yaml with your actual Argo CD URLs and passwords
kubectl apply -f argocd-credentials.yaml
```

**Option B: Using kubectl (recommended for secrets)**

```bash
# Get Argo CD admin passwords
DEV_PWD=$(kubectl --context dev-cluster get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d)
STG_PWD=$(kubectl --context staging-cluster get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d)
PROD_PWD=$(kubectl --context prod-cluster get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d)

# Create credentials in Kargo namespace
kubectl --context dev-cluster create secret generic argocd-creds-dev \
  -n kargo-advanced \
  --from-literal=url=https://argocd.dev.example.com \
  --from-literal=username=admin \
  --from-literal=password=$DEV_PWD

kubectl --context dev-cluster create secret generic argocd-creds-staging \
  -n kargo-advanced \
  --from-literal=url=https://argocd.staging.example.com \
  --from-literal=username=admin \
  --from-literal=password=$STG_PWD

kubectl --context dev-cluster create secret generic argocd-creds-prod \
  -n kargo-advanced \
  --from-literal=url=https://argocd.prod.example.com \
  --from-literal=username=admin \
  --from-literal=password=$PROD_PWD

# Label the secrets
kubectl --context dev-cluster label secret argocd-creds-dev argocd-creds-staging argocd-creds-prod \
  -n kargo-advanced \
  kargo.akuity.io/cred-type=argocd
```

### 4. Create Argo CD Applications (one per cluster)

```bash
# Apply to dev cluster
kubectl --context dev-cluster apply -f argocd-apps/dev-cluster-app.yaml

# Apply to staging cluster
kubectl --context staging-cluster apply -f argocd-apps/staging-cluster-app.yaml

# Apply to prod cluster
kubectl --context prod-cluster apply -f argocd-apps/prod-cluster-app.yaml
```

### 5. Update PromotionTask

```bash
# Replace the existing promotiontask with multi-cluster version
kubectl apply -f kargo-promotiontask.yaml
```

### 6. Create Warehouse and Stages

```bash
# Use existing warehouse and stages from parent directory
kubectl apply -f ../kargo/warehouse.yaml
kubectl apply -f ../kargo/stages.yaml
```

### 7. Verify Setup

```bash
# Check Kargo stages
kargo get stages --project kargo-advanced

# Check Argo CD credentials
kubectl get secrets -n kargo-advanced | grep argocd-creds

# Check Argo CD apps in each cluster
kubectl --context dev-cluster get applications -n argocd
kubectl --context staging-cluster get applications -n argocd
kubectl --context prod-cluster get applications -n argocd
```

## Network Connectivity

For Kargo to connect to remote Argo CD instances, you need network connectivity. Options:

### Option 1: Expose Argo CD via Ingress/LoadBalancer

**Dev Cluster:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-server-external
  namespace: argocd
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
```

Or use Ingress:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - argocd.dev.example.com
    secretName: argocd-tls
  rules:
  - host: argocd.dev.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

Repeat for staging and prod clusters with different hostnames.

### Option 2: Use NodePort (for testing)

```bash
# Expose Argo CD on each cluster
kubectl --context dev-cluster patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
kubectl --context staging-cluster patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
kubectl --context prod-cluster patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'

# Get NodePort
kubectl --context dev-cluster get svc argocd-server -n argocd -o jsonpath='{.spec.ports[0].nodePort}'
```

Then use: `https://<node-ip>:<nodeport>`

### Option 3: Port Forwarding (local testing only)

```bash
# Terminal 1: Dev cluster
kubectl --context dev-cluster port-forward -n argocd svc/argocd-server 8080:443

# Terminal 2: Staging cluster
kubectl --context staging-cluster port-forward -n argocd svc/argocd-server 8081:443

# Terminal 3: Prod cluster
kubectl --context prod-cluster port-forward -n argocd svc/argocd-server 8082:443

# Update credentials to use localhost
kubectl --context dev-cluster patch secret argocd-creds-dev -n kargo-advanced \
  --patch '{"stringData":{"url":"https://localhost:8080"}}'
```

## Promotion Flow

1. **Warehouse** detects new image (e.g., v0.0.3)
2. **Promote to dev:**
   ```bash
   kargo promote --project kargo-advanced --stage dev --freight <freight-id>
   ```
   - Kargo updates `env/dev` branch in Git
   - Kargo connects to **Argo CD in dev cluster** using `argocd-creds-dev`
   - Argo CD syncs guestbook-dev on dev cluster

3. **Promote to staging:**
   ```bash
   kargo promote --project kargo-advanced --stage staging --freight <freight-id>
   ```
   - Kargo updates `env/staging` branch in Git
   - Kargo connects to **Argo CD in staging cluster** using `argocd-creds-staging`
   - Argo CD syncs guestbook-staging on staging cluster

4. **Promote to prod:**
   ```bash
   kargo promote --project kargo-advanced --stage prod --freight <freight-id>
   ```
   - Kargo updates `env/prod` branch in Git
   - Kargo connects to **Argo CD in prod cluster** using `argocd-creds-prod`
   - Argo CD syncs guestbook-prod on prod cluster

## Troubleshooting

### Can't connect to remote Argo CD

```bash
# Test connectivity from Kargo pod
KARGO_POD=$(kubectl get pod -n kargo -l app.kubernetes.io/name=kargo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $KARGO_POD -n kargo -- curl -k https://argocd.staging.example.com
```

### Check promotion logs

```bash
# Get recent promotion
PROMOTION=$(kargo get promotion --project kargo-advanced --sort-by=.metadata.creationTimestamp -o name | tail -1)

# View logs
kubectl logs -n kargo-advanced $PROMOTION
```

### Verify credentials

```bash
kubectl get secret argocd-creds-staging -n kargo-advanced -o jsonpath='{.data.url}' | base64 -d
kubectl get secret argocd-creds-staging -n kargo-advanced -o jsonpath='{.data.username}' | base64 -d
```

## Alternative: Git-Only Approach

If you don't want Kargo to connect to remote Argo CD instances, you can:

1. Remove the `argocd-update` step from PromotionTask
2. Enable auto-sync on all Argo CD Applications
3. Argo CD will automatically sync when it detects Git changes

**Pros:** Simpler, no network connectivity needed
**Cons:** Slower (waits for Argo CD poll interval, default 3 minutes)

To enable auto-sync:
```yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Summary

Your multi-cluster setup is now complete:
- ✅ Kargo runs in one cluster (dev/management)
- ✅ Each environment has its own cluster with Argo CD
- ✅ Kargo connects to each Argo CD via credentials
- ✅ Same promotion workflow, different target clusters
- ✅ Git remains single source of truth
