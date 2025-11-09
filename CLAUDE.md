# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a GitOps repository demonstrating advanced Kargo and Argo CD integration. It showcases a multi-stage deployment pipeline that promotes both container images and Git manifest changes through dev → staging → A/B testing → production stages.

## Architecture

### Core Components

**Warehouse** (`kargo/warehouse.yaml`):
- Monitors ghcr.io container registry for new guestbook images (using SemVer strategy)
- Tracks manifest changes from the main branch
- Creates "Freight" containing both image tags and Git commits to be promoted together

**Stages** (`kargo/stages.yaml`):
- **dev**: First stage, accepts freight directly from Warehouse
- **staging**: Receives freight from dev
- **ab-test-a** and **ab-test-b**: Parallel A/B testing stages from staging with HTTP verification
- **prod**: Control flow stage requiring both A/B tests to pass
- **prod-west/central/east**: Regional production deployments from prod

**Promotion Tasks** (`kargo/promotiontasks.yaml`):
- Implements "rendered branch" pattern: clones source to `./src`, renders to `./out`
- Updates image tags in Kustomize overlays using `kustomize-set-image`
- Builds manifests with `kustomize-build` to environment-specific branches (`env/<stage>`)
- Commits and pushes rendered manifests to `env/<stage>` branches
- Syncs corresponding Argo CD Applications

**Argo CD Integration** (`argocd/appset.yaml`):
- ApplicationSet generates apps from `env/*` directories
- Each stage gets a dedicated app: `guestbook-<stage>` in namespace `guestbook-<stage>`
- Apps authorized to be synced by corresponding Kargo stage via annotation

**Verification** (`kargo/analysis.yaml`):
- A/B test stages use AnalysisTemplate `pokemon-xp` that queries PokeAPI
- Different Pokemon for each variant (pikachu vs charizard) simulate different configurations
- Verification must pass before freight qualifies for downstream promotion

### Directory Structure

```
base/                    # Base Kustomize resources (deployment, service, namespace)
env/<stage>/             # Kustomize overlays per stage (references ../../base)
kargo/                   # Kargo CRDs (warehouse, stages, promotiontasks, analysis, project)
argocd/                  # Argo CD resources (appset, appproj)
```

### Rendered Branch Pattern

This repo uses the "rendered branch" pattern:
1. Source manifests live in `base/` and `env/` on main branch
2. During promotion, Kustomize builds manifests and commits them to `env/<stage>` branches
3. Argo CD Applications sync from the `env/<stage>` branches, not main
4. This separates source-of-truth (main) from what's actually deployed (env branches)

## Setup Commands

### Initial Personalization

Before first use, personalize the repository to your GitHub username and Argo CD destination:

```bash
./personalize.sh
git commit -a -m "personalize manifests"
git push
```

This script updates all YAML files to use your GitHub username and container registry.

### Create Container Image Repository

Create a public ghcr.io image repository:

```bash
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/<your-username>/guestbook:v0.0.1
```

Then make the package public in GitHub UI to allow Kargo to monitor it without credentials.

### Install CLIs

```bash
./download-cli.sh /usr/local/bin/kargo
```

Also install `argocd` CLI and `kubectl`.

### Login to Services

```bash
kargo login https://localhost:31444/ --admin
argocd login <argocd-hostname>
```

### Deploy Argo CD Resources

```bash
argocd proj create -f ./argocd/appproj.yaml
argocd appset create ./argocd/appset.yaml
```

### Deploy Kargo Resources

```bash
kargo apply -f ./kargo
```

### Configure Git Credentials

Kargo needs write access to commit to environment branches:

```bash
kargo create credentials github-creds \
  --project kargo-advanced \
  --git \
  --username <your-username> \
  --repo-url https://github.com/<your-username>/kargo-advanced.git
```

Token must have repository write permissions.

## Development Workflow

### Promoting a New Image

Simulate a release by retagging with a new semantic version:

```bash
docker buildx imagetools create \
  ghcr.io/akuity/guestbook:latest \
  -t ghcr.io/<your-username>/guestbook:v0.0.2
```

Then refresh the Warehouse in the Kargo UI to detect new freight.

### Promoting Manifest Changes

Edit files in `base/` directory (e.g., add environment variables to `base/guestbook-deploy.yaml`), commit, and push to main. Kargo will promote these changes through the pipeline along with image updates.

### Manual Promotion

In Kargo UI:
1. Navigate to `kargo-advanced` project
2. Click target icon next to the stage you want to promote to
3. Select the freight to promote
4. Confirm promotion

Freight must pass verification before becoming available to downstream stages.

## Key Configuration Files

- `kargo/warehouse.yaml`: Defines what to monitor (images + Git)
- `kargo/stages.yaml`: Defines the promotion pipeline topology
- `kargo/promotiontasks.yaml`: Defines how promotions execute (the actual automation)
- `kargo/analysis.yaml`: Verification tests for A/B stages
- `argocd/appset.yaml`: Generates Argo CD apps for each environment
- `env/<stage>/kustomization.yaml`: Environment-specific Kustomize overlays

## Important Notes

- This repository is personalized to a specific username (`kaminirio` currently). Use `personalize.sh` to change it.
- The `prod` stage is a control flow stage with no promotion steps - it only unlocks when both A/B tests pass.
- Each promotion creates a commit on the corresponding `env/<stage>` branch.
- Argo CD Applications track the `env/<stage>` branches, not main.
- Verification uses a playful PokeAPI example but demonstrates real HTTP-based verification patterns.
