# Kargo Multi-Cluster / Multi-Argo CD Setup Guide

## Your Scenario

```
┌─────────────────────────────────────────────────────────────────┐
│ Current Setup (Single Cluster)                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────┐                       │
│  │   Kubernetes Cluster                │                       │
│  │                                     │                       │
│  │  ┌──────────┐    ┌──────────────┐  │                       │
│  │  │  Kargo   │    │   Argo CD    │  │                       │
│  │  └──────────┘    └──────────────┘  │                       │
│  │                                     │                       │
│  │  ┌──────┐  ┌─────────┐  ┌──────┐  │                       │
│  │  │ dev  │  │ staging │  │ prod │  │                       │
│  │  └──────┘  └─────────┘  └──────┘  │                       │
│  │                                     │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  All stages on same cluster                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Your Setup (Multi-Cluster)                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────┐  ┌─────────────────────┐             │
│  │   Dev Cluster       │  │  Staging Cluster    │             │
│  │                     │  │                     │             │
│  │ ┌────────────────┐  │  │ ┌────────────────┐ │   ┌──────┐  │
│  │ │ Argo CD (dev)  │  │  │ │ Argo CD (stg)  │ │   │ Prod │  │
│  │ └────────────────┘  │  │ └────────────────┘ │   │      │  │
│  │                     │  │                     │   │ ┌──┐ │  │
│  │ ┌────────────────┐  │  │ ┌────────────────┐ │   │ │CD│ │  │
│  │ │ guestbook-dev  │  │  │ │ guestbook-stg  │ │   │ └──┘ │  │
│  │ └────────────────┘  │  │ └────────────────┘ │   │ ┌──┐ │  │
│  └─────────────────────┘  └─────────────────────┘   │ │app│ │  │
│                                                      │ └──┘ │  │
│                                                      └──────┘  │
│                                                                 │
│  Each environment on separate cluster                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Options

### Option 1: Central Kargo with Multiple Argo CD Connections ⭐ **RECOMMENDED**

Kargo runs in one cluster (or separate control plane) and connects to multiple Argo CD instances.

```
┌──────────────────────────────────────────────────────────────────┐
│ Control Plane Cluster / Management Cluster                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                      Kargo                               │    │
│  │                                                          │    │
│  │  • Warehouse (monitors images/git)                      │    │
│  │  • Stages (dev, staging, prod)                          │    │
│  │  • PromotionTasks                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                       │
│                           │ Uses Argo CD credentials             │
│         ┌─────────────────┼─────────────────┐                   │
│         ↓                 ↓                 ↓                     │
└─────────┼─────────────────┼─────────────────┼────────────────────┘
          │                 │                 │
          │                 │                 │
     ┌────▼─────┐      ┌───▼──────┐     ┌───▼──────┐
     │ Dev Cluster     │ Stg Cluster    │ Prod Cluster
     │                 │                │
     │ ┌────────────┐  │ ┌────────────┐│ ┌────────────┐
     │ │ Argo CD    │  │ │ Argo CD    ││ │ Argo CD    │
     │ │ (in-cluster)  │ │ (in-cluster││ │ (in-cluster│
     │ └────────────┘  │ └────────────┘│ └────────────┘
     │                 │                │
     │ Apps deployed   │ Apps deployed  │ Apps deployed
     └─────────────────┘ └──────────────┘ └────────────┘
```

**How it works:**
1. Kargo runs in one cluster (can be dev, or separate mgmt cluster)
2. Kargo has credentials to access each Argo CD instance
3. Promotion updates Git (as usual)
4. Kargo tells the appropriate Argo CD instance to sync
5. Each Argo CD deploys to its own cluster

---

### Option 2: Git-Based Coordination (Loosely Coupled)

Kargo only updates Git; each Argo CD independently syncs.

```
┌──────────────────────────────────────────────────────────────────┐
│ Management Cluster                                                │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Kargo                                                  │     │
│  │  • Monitors images/Git                                  │     │
│  │  • Updates Git only (no Argo CD interaction)           │     │
│  └────────────────────────────────────────────────────────┘     │
│                           │                                       │
│                           ↓                                       │
│                      Updates Git                                 │
└───────────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
         Git Repo      Git Repo      Git Repo
         env/dev       env/stg       env/prod
              ↑             ↑             ↑
              │             │             │
     ┌────────┴──┐   ┌──────┴──┐   ┌─────┴────┐
     │ Dev Cluster  │ Stg Cluster  │ Prod Cluster
     │ Argo CD      │ Argo CD      │ Argo CD
     │ (auto-sync)  │ (auto-sync)  │ (auto-sync)
     └──────────────┘ └─────────────┘ └──────────┘
```

**How it works:**
1. Kargo updates Git (rendered branches or values files)
2. Each Argo CD has auto-sync enabled
3. Argo CD polls Git and syncs automatically
4. No direct Kargo → Argo CD interaction

**Pros:** Simple, no Argo CD credentials needed
**Cons:** Slower, no immediate sync confirmation

---

### Option 3: Kargo per Cluster (Distributed)

Run separate Kargo instances in each cluster (less common).

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Dev Cluster │  │ Stg Cluster │  │Prod Cluster │
│             │  │             │  │             │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │ Kargo   │ │  │ │ Kargo   │ │  │ │ Kargo   │ │
│ │ (dev)   │ │  │ │ (stg)   │ │  │ │ (prod)  │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │ ArgoCD  │ │  │ │ ArgoCD  │ │  │ │ ArgoCD  │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
└─────────────┘  └─────────────┘  └─────────────┘
```

**Cons:** Complex to coordinate, not recommended

---

## Implementation: Option 1 (Central Kargo - Recommended)

### Step 1: Install Kargo in Management/Dev Cluster

```bash
# Install Kargo in your management cluster (or dev cluster)
kubectl config use-context dev-cluster
helm install kargo oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo \
  --create-namespace
```

### Step 2: Configure Argo CD Credentials for Each Cluster

Create secrets in Kargo's namespace with Argo CD credentials:

```yaml
# argocd-creds-dev.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-creds-dev
  namespace: kargo-advanced
  labels:
    kargo.akuity.io/cred-type: argocd
type: Opaque
stringData:
  # Argo CD in dev cluster
  url: https://argocd.dev.example.com
  # OR if using in-cluster: https://argocd-server.argocd.svc.cluster.local
  username: admin
  password: <argocd-dev-password>
  # OR use token instead:
  # token: <argocd-auth-token>

---
# argocd-creds-staging.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-creds-staging
  namespace: kargo-advanced
  labels:
    kargo.akuity.io/cred-type: argocd
type: Opaque
stringData:
  url: https://argocd.staging.example.com
  username: admin
  password: <argocd-staging-password>

---
# argocd-creds-prod.yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-creds-prod
  namespace: kargo-advanced
  labels:
    kargo.akuity.io/cred-type: argocd
type: Opaque
stringData:
  url: https://argocd.prod.example.com
  username: admin
  password: <argocd-prod-password>
```

Apply credentials:
```bash
kubectl apply -f argocd-creds-dev.yaml
kubectl apply -f argocd-creds-staging.yaml
kubectl apply -f argocd-creds-prod.yaml
```

### Step 3: Create Argo CD Applications (One per Cluster)

**In Dev Cluster:**
```yaml
# On dev cluster's Argo CD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
  namespace: argocd
  annotations:
    kargo.akuity.io/authorized-stage: kargo-advanced:dev
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: env/dev
    path: .
  destination:
    server: https://kubernetes.default.svc  # Deploy to same cluster
    namespace: guestbook-dev
  project: default
```

**In Staging Cluster:**
```yaml
# On staging cluster's Argo CD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-staging
  namespace: argocd
  annotations:
    kargo.akuity.io/authorized-stage: kargo-advanced:staging
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: env/staging
    path: .
  destination:
    server: https://kubernetes.default.svc  # Deploy to same cluster
    namespace: guestbook-staging
  project: default
```

**In Prod Cluster:**
```yaml
# On prod cluster's Argo CD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
  annotations:
    kargo.akuity.io/authorized-stage: kargo-advanced:prod
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: env/prod
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-prod
  project: default
```

### Step 4: Update PromotionTask to Use Specific Argo CD Instances

The key is specifying which Argo CD instance to sync:

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: image
    value: ghcr.io/kaminirio/guestbook

  # Map stages to Argo CD instances
  - name: argocdURL
    value: |
      {{- if eq ctx.stage "dev" -}}
        https://argocd.dev.example.com
      {{- else if eq ctx.stage "staging" -}}
        https://argocd.staging.example.com
      {{- else if eq ctx.stage "prod" -}}
        https://argocd.prod.example.com
      {{- end -}}

  steps:
  # Git operations (same as before)
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        create: true

  - uses: git-clear
    config:
      path: ./out

  - uses: kustomize-set-image
    as: update-image
    config:
      path: ./src/env/${{ ctx.stage }}
      images:
      - image: ${{ vars.image }}
        tag: ${{ imageFrom(vars.image).Tag }}

  - uses: kustomize-build
    config:
      path: ./src/env/${{ ctx.stage }}
      outPath: ./out

  - uses: git-commit
    as: commit
    config:
      path: ./out
      message: ${{ task.outputs['update-image'].commitMessage }}

  - uses: git-push
    config:
      path: ./out

  # Sync specific Argo CD instance
  - uses: argocd-update
    config:
      # Specify which Argo CD instance
      url: ${{ vars.argocdURL }}
      # Reference the credentials secret
      credentialsSecret: argocd-creds-${{ ctx.stage }}
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

### Step 5: Create Kargo Stages

Stages remain the same, but each stage targets a different cluster:

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: dev
  namespace: kargo-advanced
  annotations:
    kargo.akuity.io/color: red
    # Optional: specify cluster
    kargo.akuity.io/target-cluster: dev-cluster
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: guestbook
    sources:
      direct: true
  promotionTemplate:
    spec:
      steps:
      - task:
          name: promote

---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: staging
  namespace: kargo-advanced
  annotations:
    kargo.akuity.io/color: amber
    kargo.akuity.io/target-cluster: staging-cluster
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: guestbook
    sources:
      stages:
      - dev
  promotionTemplate:
    spec:
      steps:
      - task:
          name: promote

---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: prod
  namespace: kargo-advanced
  annotations:
    kargo.akuity.io/color: blue
    kargo.akuity.io/target-cluster: prod-cluster
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: guestbook
    sources:
      stages:
      - staging
  promotionTemplate:
    spec:
      steps:
      - task:
          name: promote
```

---

## Implementation: Option 2 (Git-Based, No Argo CD Connection)

Simpler approach where Kargo only updates Git, and Argo CD auto-syncs.

### PromotionTask (Simpler - No argocd-update step)

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-gitops
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: image
    value: ghcr.io/kaminirio/guestbook

  steps:
  # Same Git operations
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        create: true

  - uses: git-clear
    config:
      path: ./out

  - uses: kustomize-set-image
    as: update-image
    config:
      path: ./src/env/${{ ctx.stage }}
      images:
      - image: ${{ vars.image }}
        tag: ${{ imageFrom(vars.image).Tag }}

  - uses: kustomize-build
    config:
      path: ./src/env/${{ ctx.stage }}
      outPath: ./out

  - uses: git-commit
    as: commit
    config:
      path: ./out
      message: ${{ task.outputs['update-image'].commitMessage }}

  - uses: git-push
    config:
      path: ./out

  # NO argocd-update step!
  # Each Argo CD will auto-sync when it detects Git changes
```

### Argo CD Applications (with Auto-Sync)

Each Argo CD must have auto-sync enabled:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: env/dev
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook-dev
  # Enable auto-sync
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**Pros:**
- No Argo CD credentials needed
- Simpler configuration
- Each cluster is independent

**Cons:**
- Slower (waits for Argo CD poll interval, default 3 minutes)
- No immediate sync confirmation
- Harder to track promotion status

---

## Accessing Multiple Argo CD Instances

### Method 1: Port Forwarding (Dev/Testing)

```bash
# Forward dev cluster Argo CD
kubectl --context dev-cluster port-forward -n argocd svc/argocd-server 8080:443 &

# Forward staging cluster Argo CD
kubectl --context staging-cluster port-forward -n argocd svc/argocd-server 8081:443 &

# Forward prod cluster Argo CD
kubectl --context prod-cluster port-forward -n argocd svc/argocd-server 8082:443 &

# Now configure Kargo to use:
# Dev:     https://localhost:8080
# Staging: https://localhost:8081
# Prod:    https://localhost:8082
```

### Method 2: Ingress (Production)

Expose each Argo CD with unique hostname:

```yaml
# dev-cluster
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
spec:
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

---
# staging-cluster
spec:
  rules:
  - host: argocd.staging.example.com
    # ...

---
# prod-cluster
spec:
  rules:
  - host: argocd.prod.example.com
    # ...
```

### Method 3: Service Mesh / VPN

Connect clusters via service mesh (Istio, Linkerd) or VPN, then use:
```
https://argocd-server.argocd.svc.cluster.local (from within same cluster)
```

---

## Complete Example: 3 Clusters Setup

### Topology

```
Control Plane (Dev Cluster):
  ├── Kargo
  ├── Warehouse (monitors ghcr.io/kaminirio/guestbook)
  └── Stages: dev, staging, prod

Dev Cluster:
  ├── Argo CD (https://argocd.dev.example.com)
  └── Application: guestbook-dev → deploys to dev cluster

Staging Cluster:
  ├── Argo CD (https://argocd.staging.example.com)
  └── Application: guestbook-staging → deploys to staging cluster

Prod Cluster:
  ├── Argo CD (https://argocd.prod.example.com)
  └── Application: guestbook-prod → deploys to prod cluster
```

### Configuration Files

**1. Argo CD Credentials (in Kargo namespace on dev cluster)**

```bash
# Create credentials for each Argo CD
kubectl create secret generic argocd-creds-dev \
  -n kargo-advanced \
  --from-literal=url=https://argocd.dev.example.com \
  --from-literal=username=admin \
  --from-literal=password=<dev-password>

kubectl label secret argocd-creds-dev \
  -n kargo-advanced \
  kargo.akuity.io/cred-type=argocd

kubectl create secret generic argocd-creds-staging \
  -n kargo-advanced \
  --from-literal=url=https://argocd.staging.example.com \
  --from-literal=username=admin \
  --from-literal=password=<staging-password>

kubectl label secret argocd-creds-staging \
  -n kargo-advanced \
  kargo.akuity.io/cred-type=argocd

kubectl create secret generic argocd-creds-prod \
  -n kargo-advanced \
  --from-literal=url=https://argocd.prod.example.com \
  --from-literal=username=admin \
  --from-literal=password=<prod-password>

kubectl label secret argocd-creds-prod \
  -n kargo-advanced \
  kargo.akuity.io/cred-type=argocd
```

**2. PromotionTask with Conditional Argo CD URLs**

See full example in Step 4 above.

**3. Deploy Applications to Each Cluster**

```bash
# Apply to dev cluster
kubectl --context dev-cluster apply -f argocd/app-dev.yaml

# Apply to staging cluster
kubectl --context staging-cluster apply -f argocd/app-staging.yaml

# Apply to prod cluster
kubectl --context prod-cluster apply -f argocd/app-prod.yaml
```

---

## Troubleshooting Multi-Cluster Setup

### Issue: Kargo can't connect to remote Argo CD

**Check credentials:**
```bash
kubectl get secret -n kargo-advanced argocd-creds-staging -o yaml
```

**Test connectivity:**
```bash
# From Kargo pod
curl -k https://argocd.staging.example.com
```

### Issue: Argo CD apps don't sync

**Check Application annotation:**
```yaml
metadata:
  annotations:
    kargo.akuity.io/authorized-stage: kargo-advanced:staging
```

**Check network connectivity between clusters**

### Issue: Promotion works but sync fails

Check Argo CD logs:
```bash
kubectl --context staging-cluster logs -n argocd deployment/argocd-application-controller
```

---

## Best Practices

1. **Use Option 1** (Central Kargo) for better visibility and control
2. **Use Git-based (Option 2)** if you want complete cluster isolation
3. **Store Argo CD credentials securely** (use Sealed Secrets or External Secrets)
4. **Use unique Git branches per environment** (env/dev, env/staging, env/prod)
5. **Monitor Kargo promotion logs** for multi-cluster issues
6. **Set up Ingress/LoadBalancer** for production Argo CD access
7. **Test connectivity** between Kargo and all Argo CD instances
8. **Use RBAC** to limit which stages can deploy to which clusters

---

## Summary

**Multi-cluster with Kargo:**
- ✅ Fully supported
- ✅ Central Kargo instance connects to multiple Argo CD
- ✅ Each cluster has its own Argo CD (in-cluster)
- ✅ Git remains single source of truth
- ✅ Same promotion workflow

**Key components:**
- One Kargo instance (control plane)
- Multiple Argo CD instances (one per cluster)
- Argo CD credentials configured in Kargo
- PromotionTask with conditional Argo CD URLs
- Applications deployed to target clusters

Your setup (dev/staging/prod on separate clusters) is a **common and well-supported pattern** in Kargo!
