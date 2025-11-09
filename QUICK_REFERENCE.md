# Quick Reference: Rollback & Skip Flow

## Your Current Pipeline

```
┌───────────────┐
│   Warehouse   │ (monitors ghcr.io/kaminirio/guestbook)
└───────┬───────┘
        │ direct
        ↓
┌───────────────┐
│      dev      │ ← Can receive freight directly from Warehouse
└───────┬───────┘
        │ sources: [dev]
        ↓
┌───────────────┐
│    staging    │ ← Requires freight to pass through dev first
└───────┬───────┘
        │ sources: [staging]
        ├─────────────────┐
        ↓                 ↓
┌──────────────┐    ┌──────────────┐
│  ab-test-a   │    │  ab-test-b   │ ← Both require staging
└──────┬───────┘    └──────┬───────┘
       │                   │
       └────────┬──────────┘
                │ sources: [ab-test-a, ab-test-b]
                ↓
        ┌───────────────┐
        │     prod      │ ← Requires BOTH A/B tests to pass
        └───────┬───────┘
                │ sources: [prod]
        ┌───────┴───────┬─────────────┐
        ↓               ↓             ↓
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │prod-west │   │prod-central│  │prod-east │
  └──────────┘   └──────────┘   └──────────┘
```

---

## Common Scenarios

### 1. Rollback `dev` to previous version

```bash
# List available freight
kargo get freight --project kargo-advanced

# Pick the freight with the old version and promote
kargo promote \
  --project kargo-advanced \
  --stage dev \
  --freight <old-freight-id>
```

**Effect:** Only `dev` rolls back. Downstream stages (staging, ab-tests, prod) are unaffected.

---

### 2. Rollback entire pipeline

```bash
# Start from the top and work down
kargo promote --project kargo-advanced --stage dev --freight <old-freight>
kargo promote --project kargo-advanced --stage staging --freight <old-freight>
kargo promote --project kargo-advanced --stage ab-test-a --freight <old-freight>
kargo promote --project kargo-advanced --stage ab-test-b --freight <old-freight>

# Prod will unlock once both ab-tests have the same freight
kargo promote --project kargo-advanced --stage prod --freight <old-freight>
kargo promote --project kargo-advanced --stage prod-west --freight <old-freight>
kargo promote --project kargo-advanced --stage prod-central --freight <old-freight>
kargo promote --project kargo-advanced --stage prod-east --freight <old-freight>
```

---

### 3. Skip to staging (bypass dev)

**Current setup:** ❌ NOT ALLOWED
- staging has `sources: stages: [dev]`
- This enforces that freight must be verified in dev first

**To allow skipping:**

Edit `kargo/stages.yaml`:
```yaml
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
      direct: true  # ← Change this
```

Apply:
```bash
kubectl apply -f kargo/stages.yaml
```

Now you can promote directly:
```bash
kargo promote --project kargo-advanced --stage staging --freight <any-freight>
```

---

### 4. Emergency hotfix to prod (skip all stages)

**Option A: Modify all stages to allow direct**
```bash
# Edit kargo/stages.yaml - change all stages to have:
#   sources:
#     direct: true

kubectl apply -f kargo/stages.yaml

# Now promote directly to prod
kargo promote --project kargo-advanced --stage prod --freight <hotfix-freight>
kargo promote --project kargo-advanced --stage prod-west --freight <hotfix-freight>
```

**Option B: Manual Freight Approval** (if supported by your Kargo version)
```bash
# Manually mark freight as approved for prod
kubectl patch freight <freight-id> -n kargo-advanced --type=merge -p '
{
  "status": {
    "approvedFor": {
      "prod": {}
    }
  }
}'

kargo promote --project kargo-advanced --stage prod --freight <freight-id>
```

**Option C: Direct Git manipulation** (emergency only)
```bash
# Get the freight's commit ID
FREIGHT_COMMIT=$(kargo get freight --project kargo-advanced <freight-id> -o yaml | grep "id:" | head -1 | awk '{print $2}')

# Manually render and push to env/prod
git checkout -b temp-hotfix
git checkout origin/main -- env/prod
cd env/prod
kustomize edit set image ghcr.io/kaminirio/guestbook:v0.0.X
kustomize build . > /tmp/prod-manifests.yaml

git checkout env/prod
rm -rf *
mv /tmp/prod-manifests.yaml .
git add .
git commit -m "Emergency hotfix: deploy v0.0.X"
git push origin env/prod --force

# Sync ArgoCD
argocd app sync guestbook-prod
```

---

### 5. Check what version is where

```bash
# Quick view of all stages
kargo get stages --project kargo-advanced

# See exact image deployed in each environment
for stage in dev staging ab-test-a ab-test-b prod-west prod-central prod-east; do
  echo "=== $stage ==="
  kubectl get application -n argocd guestbook-$stage \
    -o jsonpath='{.status.summary.images[0]}' 2>/dev/null || echo "No app"
  echo ""
done

# Check Git history for an environment
git log origin/env/dev --oneline -5
git log origin/env/staging --oneline -5
```

---

### 6. Find freight by version

```bash
# List all freight with details
kargo get freight --project kargo-advanced -o yaml | \
  grep -B 5 "tag:" | \
  grep -E "name:|tag:"

# Example output:
#   name: abc123...
#   tag: v0.0.1
#   name: def456...
#   tag: v0.0.2
```

---

## Useful Commands

```bash
# Watch promotion progress
watch -n 2 'kargo get stages --project kargo-advanced'

# Get freight ID by alias
kargo get freight --project kargo-advanced | grep "worn-fox"

# Check promotion history
kargo get promotion --project kargo-advanced

# Verify deployed image
kubectl get application -n argocd guestbook-dev \
  -o jsonpath='{.status.summary.images[0]}'

# Force Argo CD to re-sync
argocd app sync guestbook-dev --force

# Check Argo CD sync status
argocd app list -o wide | grep guestbook
```

---

## Safety Tips

✅ **Always test rollback in dev/staging first**
✅ **Keep track of freight IDs** - save them when promoting
✅ **Verify after each promotion**
✅ **Use Kargo's built-in promotion** (not direct Git) when possible

⚠️ **Skipping stages bypasses:**
- Verification checks
- Gradual rollout
- Blast radius control

🚨 **Emergency procedures** (Git manipulation):
- Should be last resort
- Document what you did
- Re-align Kargo state afterwards
