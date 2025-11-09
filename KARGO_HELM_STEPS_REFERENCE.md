# Kargo Helm Promotion Steps Reference

This document lists all Helm-related promotion steps available in Kargo.

---

## Core Helm Steps

### 1. `helm-update-chart`

Updates Helm chart version in a values file or Chart.yaml dependency.

```yaml
- uses: helm-update-chart
  as: update-chart
  config:
    path: ./src/charts/guestbook  # Path to chart or parent chart
    charts:
    - name: guestbook              # Chart name
      repository: oci://ghcr.io/kaminirio/charts
      version: ${{ chartFrom(vars.chartRepo).Version }}
```

**Use case:** Update Helm chart dependency versions

---

### 2. `helm-update-image`

Updates image tag in a Helm values file.

```yaml
- uses: helm-update-image
  as: update-image
  config:
    path: ./src/charts/guestbook           # Path to chart
    valuesFilePath: values.yaml            # Values file (default: values.yaml)
    images:
    - image: ghcr.io/kaminirio/guestbook  # Image to update
      key: image.tag                       # Path in values.yaml
      value: ${{ imageFrom(vars.image).Tag }}
```

**Output:**
- `commitMessage` - Suggested commit message

**Use case:** Update container image tags in Helm values

---

### 3. `helm-template`

Renders Helm chart to Kubernetes manifests.

```yaml
- uses: helm-template
  config:
    path: ./src/charts/guestbook     # Path to Helm chart
    outPath: ./out                   # Where to write rendered manifests
    releaseName: guestbook-dev       # Helm release name
    namespace: guestbook-dev         # Target namespace
    valuesFiles:                     # List of values files
    - ../../env/dev.yaml
    - ../../env/dev-secrets.yaml
    includeCRDs: true                # Include CRDs (default: false)
    kubeVersion: "1.28.0"            # Target Kubernetes version
    apiVersions:                     # Available API versions
    - monitoring.coreos.com/v1
```

**Use case:** Render Helm charts to plain manifests for rendered branch pattern

---

### 4. `yaml-update`

Updates YAML files (useful for Helm values files).

```yaml
- uses: yaml-update
  as: update-values
  config:
    path: ./repo/env/dev.yaml
    updates:
    - key: image.tag
      value: ${{ imageFrom(vars.image).Tag }}
    - key: replicaCount
      value: "3"
    - key: resources.limits.memory
      value: 512Mi
```

**Use case:** Update Helm values files without rendering

---

## Helper Steps Often Used with Helm

### 5. `git-clone`

Clone repository (same as Kustomize).

```yaml
- uses: git-clone
  config:
    repoURL: https://github.com/kaminirio/kargo-advanced.git
    checkout:
    - commit: ${{ commitFrom(vars.repoURL).ID }}
      path: ./src
    - branch: env/${{ ctx.stage }}
      path: ./out
      create: true
```

---

### 6. `git-commit` / `git-push`

Commit and push changes (same as Kustomize).

```yaml
- uses: git-commit
  as: commit
  config:
    path: ./repo
    message: "Deploy ${{ imageFrom(vars.image).Tag }}"

- uses: git-push
  config:
    path: ./repo
    targetBranch: main
```

---

### 7. `argocd-update`

Trigger Argo CD sync (same as Kustomize).

```yaml
- uses: argocd-update
  config:
    apps:
    - name: guestbook-${{ ctx.stage }}
      sources:
      - repoURL: ${{ vars.repoURL }}
        desiredRevision: ${{ task.outputs['commit'].commit }}
      - chart: guestbook
        repoURL: oci://ghcr.io/charts
        targetRevision: 1.2.3
```

---

## Complete Examples

### Example 1: Update Helm Chart and Image

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-helm-full
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: chartRepoURL
    value: oci://ghcr.io/kaminirio/charts
  - name: image
    value: ghcr.io/kaminirio/guestbook

  steps:
  # Clone repo
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - commit: ${{ commitFrom(vars.repoURL).ID }}
        path: ./repo

  # Update chart version
  - uses: helm-update-chart
    config:
      path: ./repo/charts/guestbook
      charts:
      - name: common-library
        repository: https://charts.bitnami.com/bitnami
        version: ${{ chartFrom(vars.chartRepoURL).Version }}

  # Update image in values
  - uses: helm-update-image
    as: update-image
    config:
      path: ./repo/charts/guestbook
      valuesFilePath: ../../env/${{ ctx.stage }}.yaml
      images:
      - image: ${{ vars.image }}
        key: image.tag
        value: ${{ imageFrom(vars.image).Tag }}

  # Commit
  - uses: git-commit
    as: commit
    config:
      path: ./repo
      message: |
        Promote ${{ ctx.stage }} to ${{ imageFrom(vars.image).Tag }}

        Chart version: ${{ chartFrom(vars.chartRepoURL).Version }}
        ${{ task.outputs['update-image'].commitMessage }}

  # Push
  - uses: git-push
    config:
      path: ./repo

  # Sync Argo CD
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        - repoURL: ${{ vars.repoURL }}
          desiredRevision: ${{ task.outputs['commit'].commit }}
```

---

### Example 2: Multi-Environment Values Update

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-multi-env
  namespace: kargo-advanced
spec:
  vars:
  - name: repoURL
    value: https://github.com/kaminirio/kargo-advanced.git
  - name: image
    value: ghcr.io/kaminirio/guestbook

  steps:
  - uses: git-clone
    config:
      repoURL: ${{ vars.repoURL }}
      checkout:
      - fromFreight: true
        path: ./repo

  # Update common values
  - uses: yaml-update
    config:
      path: ./repo/env/values-common.yaml
      updates:
      - key: app.version
        value: ${{ imageFrom(vars.image).Tag }}

  # Update stage-specific values
  - uses: yaml-update
    config:
      path: ./repo/env/${{ ctx.stage }}.yaml
      updates:
      - key: image.tag
        value: ${{ imageFrom(vars.image).Tag }}
      - key: deployedAt
        value: ${{ time() }}
      - key: deployedBy
        value: kargo

  - uses: git-commit
    as: commit
    config:
      path: ./repo
      message: "Deploy ${{ ctx.stage }} - ${{ imageFrom(vars.image).Tag }}"

  - uses: git-push
    config:
      path: ./repo

  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
```

---

### Example 3: Helm with External Chart Repository

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: PromotionTask
metadata:
  name: promote-external-chart
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
  - uses: git-clone
    config:
      repoURL: ${{ vars.gitRepoURL }}
      checkout:
      - fromFreight: true
        path: ./repo

  # Update values with new image and chart version
  - uses: yaml-update
    config:
      path: ./repo/env/${{ ctx.stage }}.yaml
      updates:
      - key: image.tag
        value: ${{ imageFrom(vars.image).Tag }}
      - key: chart.version
        value: ${{ chartFrom(vars.chartRepoURL).Version }}

  - uses: git-commit
    as: commit
    config:
      path: ./repo
      message: |
        Promote ${{ ctx.stage }}

        Image: ${{ imageFrom(vars.image).Tag }}
        Chart: ${{ chartFrom(vars.chartRepoURL).Version }}

  - uses: git-push
    config:
      path: ./repo

  # Update Argo CD with both Git and Chart sources
  - uses: argocd-update
    config:
      apps:
      - name: guestbook-${{ ctx.stage }}
        sources:
        # Chart from OCI registry
        - chart: guestbook
          repoURL: ${{ vars.chartRepoURL }}
          targetRevision: ${{ chartFrom(vars.chartRepoURL).Version }}
        # Values from Git
        - repoURL: ${{ vars.gitRepoURL }}
          targetRevision: ${{ task.outputs['commit'].commit }}
```

---

## Warehouse Configuration for Helm Charts

### Monitor Helm Chart Repository

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: guestbook
  namespace: kargo-advanced
spec:
  subscriptions:
  # Monitor Helm chart in OCI registry
  - chart:
      repoURL: oci://ghcr.io/kaminirio/charts
      name: guestbook
      semverConstraint: ^1.0.0

  # Monitor Helm chart in traditional HTTP repo
  - chart:
      repoURL: https://charts.example.com
      name: guestbook
      discoveryLimit: 20

  # Also monitor images
  - image:
      repoURL: ghcr.io/kaminirio/guestbook
      imageSelectionStrategy: SemVer

  # And Git changes
  - git:
      repoURL: https://github.com/kaminirio/kargo-advanced.git
      branch: main
```

---

## Kargo Expression Functions for Helm

### `chartFrom(repoURL)`

Get chart information from freight.

```yaml
# Usage:
${{ chartFrom(vars.chartRepoURL).Version }}   # e.g., "1.2.3"
${{ chartFrom(vars.chartRepoURL).Name }}      # e.g., "guestbook"

# Example:
- uses: yaml-update
  config:
    path: ./values.yaml
    updates:
    - key: chart.version
      value: ${{ chartFrom("oci://ghcr.io/charts").Version }}
```

### `imageFrom(repoURL)`

Get image information from freight (same as Kustomize).

```yaml
${{ imageFrom(vars.image).Tag }}      # e.g., "v0.0.2"
${{ imageFrom(vars.image).Digest }}   # e.g., "sha256:abc123..."
${{ imageFrom(vars.image).RepoURL }}  # e.g., "ghcr.io/kaminirio/guestbook"
```

### `commitFrom(repoURL)`

Get Git commit information (same as Kustomize).

```yaml
${{ commitFrom(vars.repoURL).ID }}      # e.g., "abc123def..."
${{ commitFrom(vars.repoURL).Branch }}  # e.g., "main"
${{ commitFrom(vars.repoURL).Message }} # e.g., "fix: bug"
```

---

## Common Patterns

### Pattern 1: Update Multiple Images

```yaml
- uses: helm-update-image
  config:
    path: ./charts/myapp
    valuesFilePath: ../../env/dev.yaml
    images:
    - image: ghcr.io/myorg/frontend
      key: frontend.image.tag
      value: ${{ imageFrom("ghcr.io/myorg/frontend").Tag }}
    - image: ghcr.io/myorg/backend
      key: backend.image.tag
      value: ${{ imageFrom("ghcr.io/myorg/backend").Tag }}
    - image: ghcr.io/myorg/worker
      key: worker.image.tag
      value: ${{ imageFrom("ghcr.io/myorg/worker").Tag }}
```

### Pattern 2: Conditional Values per Stage

```yaml
- uses: yaml-update
  config:
    path: ./env/${{ ctx.stage }}.yaml
    updates:
    - key: image.tag
      value: ${{ imageFrom(vars.image).Tag }}
    - key: replicaCount
      value: ${{ ctx.stage == "prod" ? "3" : "1" }}
    - key: resources.limits.memory
      value: ${{ ctx.stage == "prod" ? "2Gi" : "512Mi" }}
```

### Pattern 3: Render with Multiple Values Files

```yaml
- uses: helm-template
  config:
    path: ./charts/guestbook
    outPath: ./rendered
    releaseName: guestbook-${{ ctx.stage }}
    namespace: guestbook-${{ ctx.stage }}
    valuesFiles:
    - values.yaml                    # Base values
    - ../../env/common.yaml          # Common overrides
    - ../../env/${{ ctx.stage }}.yaml # Stage-specific
```

---

## Migration Checklist

Moving from Kustomize to Helm:

- [ ] Create Helm chart structure
- [ ] Convert manifests to templates
- [ ] Create values.yaml with parameterized values
- [ ] Create environment-specific values files
- [ ] Update PromotionTask to use Helm steps
- [ ] Choose pattern (rendered branch vs values file)
- [ ] Update Argo CD ApplicationSet
- [ ] Test promotion in dev
- [ ] Update CLAUDE.md with new structure
- [ ] Remove old Kustomize files

---

## Troubleshooting

### Issue: Helm template step fails

```yaml
# Add debug output
- uses: helm-template
  config:
    path: ./charts/myapp
    valuesFiles:
    - values.yaml
    validate: false  # Skip schema validation
    # OR
    debug: true      # Show debug output
```

### Issue: Values not updating

```yaml
# Verify the key path matches your values structure
# values.yaml:
#   image:
#     repository: foo
#     tag: bar
#
# Use dot notation:
- key: image.tag  # Correct
# NOT
- key: image/tag  # Wrong
```

### Issue: Chart dependencies not updated

```yaml
# Make sure to update Chart.yaml dependencies
- uses: helm-update-chart
  config:
    path: ./charts/myapp
    charts:
    - name: common
      repository: https://charts.bitnami.com/bitnami
      version: ${{ chartFrom(vars.chartRepo).Version }}
```

---

## Best Practices

1. **Use Values File Pattern** unless you need exact manifest auditing
2. **Keep values files simple** - use one per environment
3. **Version your charts** with semantic versioning
4. **Test locally** before promoting:
   ```bash
   helm template my-release ./charts/myapp -f env/dev.yaml
   ```
5. **Use chart repositories** (OCI or HTTP) for reusable charts
6. **Document values** in values.yaml with comments
7. **Validate charts** in CI/CD before Kargo promotion

---

## Additional Resources

- [Kargo Documentation](https://kargo.akuity.io)
- [Helm Documentation](https://helm.sh/docs/)
- [Argo CD Helm Integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
