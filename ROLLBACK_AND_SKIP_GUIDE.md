# Kargo Rollback and Direct Deployment Guide

## Method 1: Rollback by Re-Promoting Previous Freight

### Step 1: List all freight to find the version you want
```bash
kargo get freight --project kargo-advanced
```

Output example:
```
NAME                                       ALIAS                 ORIGIN                AGE
ab49300...  (v0.0.2)                       worn-fox              Warehouse/guestbook   1h
xyz12345... (v0.0.1)                       happy-cat             Warehouse/guestbook   5h
```

### Step 2: Check what's currently deployed
```bash
kargo get stages --project kargo-advanced
```

### Step 3: Promote the older freight to rollback
```bash
# Rollback dev to v0.0.1
kargo promote \
  --project kargo-advanced \
  --stage dev \
  --freight xyz12345...  # Use the freight ID with v0.0.1

# Rollback staging to previous version
kargo promote \
  --project kargo-advanced \
  --stage staging \
  --freight xyz12345...
```

**What happens:**
- Kargo runs the promotion task again with the OLD freight
- Updates env/dev branch with the old image version
- Argo CD syncs and deploys the old version
- All changes are tracked in Git (new commit on env branch)

---

## Method 2: Skip Stages and Deploy Directly

### Scenario: You want to deploy v0.0.3 directly to `staging`, skipping `dev`

**IMPORTANT:** By default, Kargo enforces the pipeline flow. You need to check stage configuration:

```bash
kubectl get stage staging -n kargo-advanced -o yaml
```

Look for `requestedFreight.sources`:
```yaml
requestedFreight:
- origin:
    kind: Warehouse
    name: guestbook
  sources:
    stages:
    - dev  # ← This means staging REQUIRES freight from dev
```

### Option 2A: Temporary - Manually Approve Freight

If a stage requires upstream approval, you can manually approve freight:

```bash
# Manually approve freight for staging (bypassing dev)
kargo approve \
  --project kargo-advanced \
  --stage staging \
  --freight ab49300...
```

Then promote:
```bash
kargo promote \
  --project kargo-advanced \
  --stage staging \
  --freight ab49300...
```

### Option 2B: Modify Stage to Allow Direct Promotion

Edit the stage to accept freight directly from Warehouse:

```yaml
# kargo/stages.yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: staging
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: guestbook
    sources:
      direct: true  # ← Changed from stages: [dev]
```

Apply:
```bash
kubectl apply -f kargo/stages.yaml
```

Now you can promote directly:
```bash
kargo promote \
  --project kargo-advanced \
  --stage staging \
  --freight <any-freight-id>
```

**WARNING:** This bypasses the pipeline and verification!

---

## Method 3: Emergency Rollback via Git

### When Kargo is down or you need immediate rollback:

```bash
# 1. Check what commit was previously deployed
git log origin/env/staging --oneline

# Output:
# abc1234 Updated to use v0.0.3  ← Current (bad)
# def5678 Updated to use v0.0.2  ← Previous (good)

# 2. Reset env branch to previous commit
git checkout env/staging
git reset --hard def5678
git push --force origin env/staging

# 3. Trigger Argo CD sync
argocd app sync guestbook-staging
```

**CAUTION:** This bypasses Kargo's tracking! Kargo will still think the stage has the new freight.

To fix Kargo's state afterwards:
```bash
# Re-promote the correct freight to align Kargo's state
kargo promote \
  --project kargo-advanced \
  --stage staging \
  --freight def5678...
```

---

## Method 4: Deploy Specific Version (Create New Freight)

### If the version you want doesn't exist as freight:

```bash
# 1. Tag the image version you want
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/kaminirio/guestbook:v0.0.1

# 2. Refresh the Warehouse to discover it
# In Kargo UI: Click "Refresh" on the Warehouse
# OR via kubectl:
kubectl annotate warehouse guestbook \
  -n kargo-advanced \
  kargo.akuity.io/refresh="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --overwrite

# 3. Wait for new freight to be created
kargo get freight --project kargo-advanced

# 4. Promote the new freight
kargo promote \
  --project kargo-advanced \
  --stage dev \
  --freight <new-freight-id>
```

---

## Best Practices

### ✅ Recommended Rollback Method
**Use Method 1** (re-promote previous freight) because:
- Fully tracked by Kargo
- Creates audit trail in Git
- Maintains GitOps principles
- Can be automated

### ⚠️ Use with Caution
**Method 2B** (skip stages) should only be used for:
- Emergency hotfixes
- Testing in non-production
- Breaking the glass scenarios

### 🚨 Emergency Only
**Method 3** (direct Git manipulation) only for:
- Kargo is completely down
- Immediate production incident
- Always re-align Kargo state afterwards

---

## Verification After Rollback

```bash
# Check stage status
kargo get stages --project kargo-advanced

# Check deployed version
kubectl get application -n argocd guestbook-dev \
  -o jsonpath='{.status.summary.images[0]}'

# Check Git history
git log origin/env/dev --oneline -5
```

---

## Example: Complete Rollback Workflow

```bash
# 1. Something is wrong in staging with v0.0.3
# 2. List freight to find v0.0.2
kargo get freight --project kargo-advanced

# 3. Find the freight with v0.0.2
# NAME: abc123... ALIAS: old-freight

# 4. Rollback staging
kargo promote \
  --project kargo-advanced \
  --stage staging \
  --freight abc123...

# 5. Monitor the rollback
watch kargo get stages --project kargo-advanced

# 6. Verify
kubectl get application -n argocd guestbook-staging \
  -o jsonpath='{.status.summary.images[0]}'
# Should show: ghcr.io/kaminirio/guestbook:v0.0.2
```

Done! You've successfully rolled back to a previous version while maintaining full GitOps compliance.
