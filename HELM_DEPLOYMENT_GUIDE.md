# Kargo with Helm Deployment Guide

## Overview

Kargo supports Helm deployments with similar patterns to Kustomize. There are **two main approaches**:

1. **Rendered Branch Pattern** (like current Kustomize setup)
2. **Values File Pattern** (Helm-specific, no rendering)

---

## Approach 1: Helm with Rendered Branch Pattern

This is similar to your current Kustomize setup - render Helm charts to manifests and commit them.

### Directory Structure

```
repo/
├── charts/
│   └── guestbook/           # Helm chart source
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
├── env/
│   ├── dev/
│   │   └── values.yaml      # Environment-specific values
│   ├── staging/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
└── kargo/
    └── promotiontasks.yaml
```

### PromotionTask Example

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-helm
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: image
    value: ghcr.io/kaminirio/guestbook
  - name: chartPath
    value: ./src/charts/guestbook

  steps:
  # 1. Clone source repo and env branch
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./src
      - branch: env/${{ ctx.stage }}
        path: ./out
        create: true

  # 2. Clear env branch (rendered branch pattern)
  - uses: git-clear
    config:
      path: ./out

  # 3. Update image version in Helm values
  - uses: helm-update-image
    as: update-image
    config:
      path: ${{ vars.chartPath }}
      valuesFilePath: ../../../env/${{ ctx.stage }}/values.yaml
      images:
      - image: ${{ vars.image }}
        key: image.tag  # Path in values.yaml
        value: ${{ imageFrom(vars.image).Tag }}

  # 4. Render Helm chart to manifests
  - uses: helm-template
    config:
      path: ${{ vars.chartPath }}
      valuesFiles:
      - ../../../env/${{ ctx.stage }}/values.yaml
      outPath: ./out
      releaseName: guestbook-${{ ctx.stage }}
      namespace: guestbook-${{ ctx.stage }}

  # 5. Commit rendered manifests
  - uses: git-commit
    as: commit
    config:
      path: ./out
      message: |
        Updated ${{ ctx.stage }} to use image ${{ imageFrom(vars.image).Tag }}

        Chart: guestbook
        Stage: ${{ ctx.stage }}

  # 6. Push to env branch
  - uses: git-push
    config:
      path: ./out

  # 7. Sync Argo CD
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

### Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: env/dev  # Points to rendered branch
    path: .                   # Rendered manifests at root
  destination:
    namespace: guestbook-dev
    server: https://kubernetes.default.svc
```

**Pros:**
- ✅ Git contains exact manifests deployed (full GitOps)
- ✅ Easy to audit what's deployed
- ✅ No Helm required in cluster
- ✅ Similar to current Kustomize workflow

**Cons:**
- ❌ Larger Git history (binary data from rendering)
- ❌ Can't easily see what values changed
- ❌ No Helm release history in cluster

---

## Approach 2: Helm Values File Pattern (Recommended)

This is more Helm-native - update values files and let Argo CD render at sync time.

### Directory Structure

```
repo/
├── charts/
│   └── guestbook/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── env/
│   ├── dev.yaml          # Values file per environment
│   ├── staging.yaml
│   └── prod.yaml
└── kargo/
    └── promotiontasks.yaml
```

### PromotionTask Example

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-helm-values
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: image
    value: ghcr.io/kaminirio/guestbook

  steps:
  # 1. Clone repo
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - fromFreight: true  # Clone at the freight's commit
        path: ./repo

  # 2. Update image tag in values file
  - uses: yaml-update
    as: update-image
    config:
      path: ./repo/env/${{ ctx.stage }}.yaml
      updates:
      - key: image.tag
        value: ${{ imageFrom(vars.image).Tag }}
      - key: image.repository
        value: ${{ imageFrom(vars.image).RepoURL }}

  # 3. Commit changes
  - uses: git-commit
    as: commit
    config:
      path: ./repo
      message: |
        Promote ${{ ctx.stage }} to ${{ imageFrom(vars.image).Tag }}

  # 4. Push changes
  - uses: git-push
    config:
      path: ./repo
      targetBranch: main  # Push to main branch (not env branch!)

  # 5. Wait for Argo CD to sync
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

### Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
spec:
  source:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: main              # Points to main branch
    path: charts/guestbook            # Path to Helm chart
    helm:
      valueFiles:
      - ../../env/dev.yaml            # Environment values
  destination:
    namespace: guestbook-dev
    server: https://kubernetes.default.svc
```

### Example Values File (env/dev.yaml)

```yaml
# env/dev.yaml
replicaCount: 1

image:
  repository: ghcr.io/kaminirio/guestbook
  tag: v0.0.2  # ← Kargo updates this
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

**Pros:**
- ✅ Smaller Git diffs (just values changed)
- ✅ Helm releases tracked in cluster
- ✅ Easy to see what changed
- ✅ Can use `helm rollback` in cluster
- ✅ More Helm-native

**Cons:**
- ❌ Need to render locally to see exact manifests
- ❌ Argo CD does the rendering (deployment-time rendering)

---

## Approach 3: Helm Chart Repository Pattern

Use Helm charts from a chart repository (like ChartMuseum, Harbor, or OCI registry).

### PromotionTask Example

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-helm-chart
  namespace: kargo-advanced
spec:
  vars:
  - name: gitRepoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: chartRepoURL
    value: oci://ghcr.io/kaminirio/charts
  - name: image
    value: ghcr.io/kaminirio/guestbook

  steps:
  # 1. Clone Git repo (for values files)
  - uses: git-clone
    config:
      repoURL: ${{ vars.gitRepoURL }}
      checkout:
      - fromFreight: true
        path: ./repo

  # 2. Update chart version in values
  - uses: yaml-update
    config:
      path: ./repo/env/${{ ctx.stage }}.yaml
      updates:
      - key: image.tag
        value: ${{ imageFrom(vars.image).Tag }}
      - key: chart.version
        value: ${{ chartFrom(vars.chartRepoURL).Version }}

  # 3. Commit and push
  - uses: git-commit
    as: commit
    config:
      path: ./repo
      message: "Promote ${{ ctx.stage }} to chart ${{ chartFrom(vars.chartRepoURL).Version }}"

  - uses: git-push
    config:
      path: ./repo

  # 4. Update Argo CD
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.chartRepoURL }}
          chart: guestbook
          targetRevision: ${{ chartFrom(vars.chartRepoURL).Version }}
```

### Argo CD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-dev
spec:
  sources:
  # Source 1: Helm chart from OCI registry
  - repoURL: oci://ghcr.io/kaminirio/charts
    chart: guestbook
    targetRevision: 1.0.0  # Chart version
    helm:
      valueFiles:
      - $values/env/dev.yaml
  # Source 2: Values from Git
  - repoURL: https://github.com/kaminirio/kargo-advanced.git
    targetRevision: main
    ref: values
  destination:
    namespace: guestbook-dev
    server: https://kubernetes.default.svc
```

---

## Warehouse Configuration for Helm

### Monitor Helm Chart Repository

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: guestbook
  namespace: kargo-advanced
spec:
  subscriptions:
  # Monitor Helm chart versions
  - chart:
      repoURL: oci://ghcr.io/kaminirio/charts
      name: guestbook
      semverConstraint: ^1.0.0  # Only 1.x.x versions

  # Monitor container image
  - image:
      repoURL: ghcr.io/kaminirio/guestbook
      imageSelectionStrategy: SemVer
      allowTags: ^v[0-9]+\.[0-9]+\.[0-9]+$

  # Monitor Git repo
  - git:
      repoURL: https://github.com/kaminirio/kargo-advanced.git
      branch: main
```

---

## Comparison Table

| Feature | Kustomize (Current) | Helm Rendered | Helm Values | Helm Chart Repo |
|---------|---------------------|---------------|-------------|-----------------|
| Git has manifests | ✅ | ✅ | ❌ | ❌ |
| Small Git diffs | ❌ | ❌ | ✅ | ✅ |
| Helm releases | ❌ | ❌ | ✅ | ✅ |
| Cluster-side rendering | ❌ | ❌ | ✅ | ✅ |
| Easy rollback | ✅ | ✅ | ✅ | ✅ |
| Complex logic | ❌ | ✅ | ✅ | ✅ |
| Templating | Limited | ✅ | ✅ | ✅ |

---

## Migration Path: Kustomize → Helm

### Step 1: Create Helm Chart from Existing Manifests

```bash
# Create chart structure
mkdir -p charts/guestbook/templates

# Move existing manifests to templates
cp base/guestbook-deploy.yaml charts/guestbook/templates/deployment.yaml
cp base/guestbook-svc.yaml charts/guestbook/templates/service.yaml
cp base/guestbook-ns.yaml charts/guestbook/templates/namespace.yaml

# Create Chart.yaml
cat > charts/guestbook/Chart.yaml << EOF
apiVersion: v2
name: guestbook
description: Guestbook application
type: application
version: 1.0.0
appVersion: "v0.0.2"
EOF

# Create values.yaml
cat > charts/guestbook/values.yaml << EOF
image:
  repository: ghcr.io/kaminirio/guestbook
  tag: v0.0.2
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 3000
EOF
```

### Step 2: Template the Deployment

Edit `charts/guestbook/templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  template:
    spec:
      containers:
      - name: guestbook
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.port }}
```

### Step 3: Create Environment Values

```bash
# env/dev.yaml
cat > env/dev.yaml << EOF
replicaCount: 1
image:
  tag: v0.0.2
EOF

# env/staging.yaml
cat > env/staging.yaml << EOF
replicaCount: 2
image:
  tag: v0.0.2
EOF

# env/prod-west.yaml
cat > env/prod-west.yaml << EOF
replicaCount: 3
image:
  tag: v0.0.2
EOF
```

### Step 4: Update PromotionTask

Replace `kargo/promotiontasks.yaml` with Helm-based version (see Approach 2 above).

### Step 5: Update Argo CD Applications

```bash
# Update ApplicationSet
kubectl apply -f - << EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - stage: dev
      - stage: staging
      - stage: ab-test-a
      - stage: ab-test-b
      - stage: prod-west
      - stage: prod-central
      - stage: prod-east
  template:
    metadata:
      name: guestbook-{{stage}}
    spec:
      source:
        repoURL: https://github.com/kaminirio/kargo-advanced.git
        targetRevision: main
        path: charts/guestbook
        helm:
          valueFiles:
          - ../../env/{{stage}}.yaml
      destination:
        namespace: guestbook-{{stage}}
        server: https://kubernetes.default.svc
      project: guestbook
EOF
```

---

## Key Differences: Kustomize vs Helm

### Your Current Kustomize Flow:
```
1. git-clone (main → ./src, env/dev → ./out)
2. git-clear (./out)
3. kustomize-set-image (update kustomization.yaml)
4. kustomize-build (render to ./out)
5. git-commit (./out)
6. git-push (env/dev branch)
7. argocd-update (sync from env/dev branch)
```

### Helm Values Flow (Recommended):
```
1. git-clone (main → ./repo)
2. yaml-update (update env/dev.yaml)
3. git-commit (./repo)
4. git-push (main branch)
5. argocd-update (sync from main, render at deploy time)
```

### Helm Rendered Flow (Similar to Current):
```
1. git-clone (main → ./src, env/dev → ./out)
2. git-clear (./out)
3. helm-update-image (update values.yaml)
4. helm-template (render to ./out)
5. git-commit (./out)
6. git-push (env/dev branch)
7. argocd-update (sync from env/dev branch)
```

---

## Recommendation

For your use case, I recommend **Approach 2: Helm Values File Pattern** because:

1. **Smaller Git history** - Only values change, not full manifests
2. **Better visibility** - Easy to see what values changed per environment
3. **Helm-native** - Use Helm's features (hooks, tests, rollbacks)
4. **Simpler pipeline** - No need for rendered branches
5. **Faster promotions** - Less Git operations

The rendered branch pattern (Approach 1) is useful if you want exact manifest auditing, but adds complexity and Git bloat.
