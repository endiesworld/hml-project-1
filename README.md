# hml-project-1 (Sochi Homelab GitOps)

Production GitOps repository for the Sochi homelab Kubernetes cluster (K3s + FluxCD + Kustomize).

This repo is the **single source of truth** for cluster application state in the `staging` environment.

## Contents

- [Overview](#overview)
- [Repository layout](#repository-layout)
- [How Flux applies changes](#how-flux-applies-changes)
- [Prerequisites](#prerequisites)
- [Common operations](#common-operations)
- [Adding a new app](#adding-a-new-app)
- [Releasing, verifying, rollback](#releasing-verifying-rollback)
- [Secrets and private images](#secrets-and-private-images)
- [Conventions](#conventions)
- [Troubleshooting](#troubleshooting)

## Overview

This repository exists to:

- Manage Kubernetes workloads via **GitOps** (Flux watches this repo and reconciles desired state).
- Keep app deployment configuration separate from app source code and CI/CD pipelines.
- Support multiple apps/projects in one cluster using a Kustomize base/overlay structure.

GitOps model:

- **Application repos** contain application source, `Dockerfile`, and the image build/push pipeline.
- **This repo** contains Kubernetes manifests (Kustomize) and Flux reconciliation config.

Typical release flow:

1. Update app code in the app repo
2. Build/push a new immutable image tag (e.g., `v1.2.3`)
3. Update the image reference in this repo
4. Push the GitOps change
5. Flux reconciles and deploys automatically

## Repository layout

```text
.
├── apps/
│   ├── base/
│   │   └── <app-name>/               # shared manifests for an app
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── kustomization.yaml
│   │       └── (optional) namespace.yaml, storage.yaml, etc.
│   └── staging/
│       ├── kustomization.yaml        # aggregates all staging apps (see below)
│       └── <app-name>/               # staging overlay for an app
│           ├── kustomization.yaml
│           └── (optional) namespace.yaml, ingress.yaml, cloudflare.yaml, etc.
└── clusters/
    └── staging/
        ├── apps.yaml                 # Flux Kustomization entrypoint (path: ./apps/staging)
        └── flux-system/              # Flux bootstrap manifests
```

## How Flux applies changes

`clusters/staging/apps.yaml` defines the Flux `Kustomization` that reconciles the staging workloads:

```yaml
spec:
  path: ./apps/staging
```

That means `apps/staging` must be a valid Kustomize root. In practice, this is an aggregator file that lists all app overlays:

```yaml
# apps/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./linkding
  # - ./another-app
```

If `apps/staging/kustomization.yaml` is missing, Flux will fail to build `./apps/staging` until you add it.

## Prerequisites

- A running K3s cluster reachable from your workstation (`kubectl` works).
- Flux is bootstrapped and pointing at this repository (manifests live under `clusters/staging/flux-system`).
- Tooling installed locally: `kubectl`, `flux`, and `kustomize` (or `kubectl kustomize`).
- Registry auth configured if you deploy private images (via `imagePullSecrets`).

## Common operations

Reconcile and check status:

```bash
flux get kustomizations -A
flux reconcile source git flux-system
flux reconcile kustomization apps -n flux-system --with-source
kubectl get pods -A
```

Validate manifests locally before pushing (recommended):

```bash
kubectl kustomize apps/staging | kubectl apply --dry-run=client -f -
```

## Adding a new app

Example app name: `fleetmindai`.

1) Create base + overlay directories:

```bash
mkdir -p apps/base/fleetmindai apps/staging/fleetmindai
```

2) Add base manifests under `apps/base/fleetmindai/`:

- `deployment.yaml`
- `service.yaml`
- `kustomization.yaml`
- optional: `namespace.yaml`, `storage.yaml`, etc.

3) Add the staging overlay `apps/staging/fleetmindai/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: fleetmindai
resources:
  - ../../base/fleetmindai
  # - namespace.yaml
  # - ingress.yaml
  # - cloudflare.yaml
```

4) Register the overlay in `apps/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./linkding
  - ./fleetmindai
```

5) Commit and deploy:

```bash
git add .
git commit -m "feat: add fleetmindai (staging)"
git push
flux reconcile source git flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

## Releasing, verifying, rollback

Update an existing app release:

1) Push a new image tag from the app repo (example: `ghcr.io/endiesworld/fleetmindai:v0.1.1`).
2) Update the image reference in the manifests in this repo (typically `apps/base/<app>/deployment.yaml` or via overlay patches).
3) Commit and reconcile:

```bash
git add .
git commit -m "release(fleetmindai): bump image to v0.1.1"
git push
flux reconcile source git flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

Verify rollout:

```bash
kubectl -n fleetmindai get deploy,pods,svc
kubectl -n fleetmindai rollout status deploy/fleetmindai
kubectl -n fleetmindai describe pod <pod-name> | grep -E "Image:|Image ID:"
```

Rollback (GitOps-safe):

Preferred rollback is to revert the Git change:

```bash
git revert <commit_that_bumped_image>
git push
flux reconcile source git flux-system
flux reconcile kustomization apps -n flux-system --with-source
```

## Secrets and private images

- Do not commit plaintext secrets to Git.
- Prefer an encrypted secret workflow (e.g., SOPS + age) for production-grade secret management.

For private images, create an `imagePullSecret` in the target namespace and reference it from the `Deployment`:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-creds
```

## Conventions

- Use immutable image tags (`vX.Y.Z`) and avoid `latest` for application images.
- Keep one folder per app under `apps/base/` and `apps/staging/`.
- Keep app source repos and this GitOps repo separate.
- Every deploy-worthy change should be reproducible from Git history.

## Troubleshooting

Quick commands:

```bash
flux get all -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl -n <ns> describe pod <pod>
kubectl -n <ns> logs deploy/<name> --tail=200
kubectl kustomize apps/staging
```

Common issues:

- **ImagePullBackOff**: wrong tag, missing pull secret, or registry auth.
- **Wrong namespace**: check `namespace:` in the overlay and any namespace resources.
- **Kustomize errors**: missing `kustomization.yaml` or incorrect resource paths.
