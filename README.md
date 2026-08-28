# gitops-multienv-bootstrap

ArgoCD app-of-apps pattern for managing multiple services across multiple environments on EKS.

Built from the same pattern used at SurveyMonkey to migrate 7+ EKS environments from Terraform-managed Helm releases to GitOps — reducing deployment lead time and eliminating configuration drift.

## The Problem with Terraform for App Deployments

```
Before (Terraform managing Helm releases):
  terraform plan   → 45-second plan just to bump an image tag
  terraform apply  → change is applied but there's no live reconciliation
  manual drift     → someone kubectl edits a deployment → Terraform out of sync
  state contention → two engineers running apply at the same time → lock errors
```

```
After (ArgoCD app-of-apps):
  git push         → ArgoCD detects the change in < 3 minutes and syncs
  selfHeal=true    → manual kubectl edits are automatically reverted
  no locks         → ArgoCD is the single source of truth, no state files
  audit trail      → every change is a git commit
```

## Architecture

```
                     Git Repository
                          │
                          │  git push → ArgoCD detects change
                          ▼
                   ┌─────────────┐
                   │  Root App   │  (bootstrap/root-app.yaml — applied once)
                   │  watches    │
                   │  apps/dev/  │
                   └──────┬──────┘
                          │  creates child Applications
              ┌───────────┼───────────┐
              ▼           ▼           ▼
     api-service-dev   worker-dev   (any new .yaml added to apps/dev/)
              │           │
              │  each Application points to a Helm chart
              ▼           ▼
     charts/api-service  charts/worker
              │
              │  with layered values:
              ▼
     values.yaml (base) + values-dev.yaml (overrides)
```

## Environment Value Layers

```
charts/api-service/
  values.yaml          ← base: image, ports, defaults
  values-dev.yaml      ← dev: replicas=1, minimal resources, HPA off
  values-staging.yaml  ← staging: replicas=2, HPA on (2-5)
  values-prod.yaml     ← prod: replicas=3, full resources, HPA on (3-20)
```

ArgoCD merges these in order: base → environment override. Later values win.

## Repository Structure

```
gitops-multienv-bootstrap/
├── bootstrap/
│   ├── root-app.yaml       # apply once to bootstrap a cluster
│   └── appset.yaml         # alternative: ApplicationSet for multi-cluster
├── apps/
│   ├── dev/
│   │   ├── api-service.yaml
│   │   └── worker.yaml
│   ├── staging/
│   │   ├── api-service.yaml
│   │   └── worker.yaml
│   └── prod/
│       ├── api-service.yaml
│       └── worker.yaml
└── charts/
    ├── api-service/        # HTTP service: Deployment + Service + HPA
    └── worker/             # Background worker: Deployment + HPA
```

## Bootstrap a Cluster

### Option A — App-of-Apps (one cluster, one environment)

```bash
# 1. Install ArgoCD on the cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Edit bootstrap/root-app.yaml:
#    Set `spec.source.path` to: apps/dev, apps/staging, or apps/prod
#    Set `spec.source.repoURL` to this repo's URL

# 3. Apply the root Application
kubectl apply -f bootstrap/root-app.yaml

# 4. ArgoCD creates all child Applications from apps/<env>/
#    and syncs them automatically
```

### Option B — ApplicationSet (multi-cluster hub)

```bash
# 1. Register all clusters as ArgoCD cluster secrets
# 2. Update cluster URLs in bootstrap/appset.yaml
# 3. Apply the ApplicationSet to the hub cluster
kubectl apply -n argocd -f bootstrap/appset.yaml
# ArgoCD generates one Application per (env × app) combination
```

## Add a New Service

```bash
# 1. Create the Helm chart
cp -r charts/api-service charts/my-new-service
# edit charts/my-new-service/Chart.yaml and values files

# 2. Add Application manifests
cp apps/dev/api-service.yaml apps/dev/my-new-service.yaml
# edit: name, path (charts/my-new-service), valueFiles

# Same for staging and prod

# 3. git push
# ArgoCD picks up the new Application files automatically
```

## Add a New Environment

```bash
# 1. Add value override files to each chart
cp charts/api-service/values-staging.yaml charts/api-service/values-uat.yaml
# edit resource sizes and HPA settings

# 2. Create the apps directory
mkdir apps/uat
cp apps/staging/*.yaml apps/uat/
# edit: namespace, valueFiles to values-uat.yaml

# 3. git push
# If using app-of-apps: update root-app.yaml path on the new cluster
# If using ApplicationSet: add the uat element to bootstrap/appset.yaml
```

## Test Charts Locally

```bash
# Lint all charts
helm lint charts/api-service
helm lint charts/api-service -f charts/api-service/values-prod.yaml

# Render and inspect templates
helm template api-service-dev charts/api-service \
  -f charts/api-service/values-dev.yaml

# Dry-run against a real cluster
helm install api-service-dev charts/api-service \
  -f charts/api-service/values-dev.yaml \
  -n dev --create-namespace --dry-run
```

## App-of-Apps vs ApplicationSet

| | App-of-Apps | ApplicationSet |
|---|---|---|
| **How it works** | Root app watches a directory of Application YAMLs | Generator creates Applications dynamically |
| **Add a service** | Write a new YAML in `apps/<env>/` | Add to the generator list |
| **Add an environment** | New `apps/<env>/` directory + update root-app per cluster | Add one element to the generator list |
| **Best for** | Simple setups, full control over each Application | Many services × many clusters; DRY config |
| **This repo shows** | `bootstrap/root-app.yaml` | `bootstrap/appset.yaml` |

## Key Design Decisions

**`selfHeal: true`** — ArgoCD reverts any manual `kubectl` changes within ~3 minutes. This is intentional: the git repo is the only source of truth. If you need to make an emergency change, make it in git.

**`prune: true`** — Deleting an Application YAML from git removes the deployed resources. This prevents ghost resources that nobody owns.

**`finalizers: resources-finalizer`** — Deleting the ArgoCD Application object cascades to delete all the Kubernetes resources it manages. Without this, deleting the Application leaves the Deployment/Service behind.

**Value layering** — `values.yaml` is the contract for what a chart accepts. Environment files only override what differs. This keeps diffs small and makes cross-environment comparison easy.
