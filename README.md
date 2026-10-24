# Sync Seat GitOps for EKS

This repository is the GitOps control plane for the Sync Seat application stack. It does not contain the application business logic itself; instead, it defines the desired Kubernetes state for the platform using Helm and Argo CD. The deployment model is designed to keep runtime configuration declarative, versioned, and continuously reconciled from Git.

The application source repository is the application codebase for the Sync Seat services and frontend, while this repository manages the cluster deployment model, environment overlays, and release automation.

## Why this repository exists

This repo implements a GitOps workflow for the Sync Seat platform in Kubernetes:

- Infrastructure and workloads are declared in Git.
- Helm charts package each component and its runtime defaults.
- Environment-specific values are separated between development and production.
- Argo CD continuously reconciles cluster state to match the Git repository.
- GitHub Actions validates the Helm charts before deployment.
- Automated sync and self-healing help maintain drift-free infrastructure.

This gives the team a clean separation between:

- application source code
- container images
- Kubernetes manifests
- environment configuration
- cluster reconciliation logic

## GitOps architecture

The deployment pattern in this repository follows a standard GitOps flow:

```text
Developer / CI
    |
    v
Application source repo
    |
    |  build + push container images
    v
GitHub Actions (validation)
    |
    v
This GitOps repo (Helm + ArgoCD manifests)
    |
    v
Argo CD on Kubernetes cluster
    |
    v
Namespace: sync-seat
    |
    +--> Postgres
    +--> Redis
    +--> Booking service
    +--> Mail service
    +--> Homepage service
    +--> API Gateway
    +--> Frontend
```

## Repository layout

```text
.
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions validation workflow
├── argocd/
│   ├── application.yaml            # Argo CD Application definition
│   └── project.yaml                 # Argo CD AppProject permissions and scope
├── helm/
│   └── syncseat/
│       ├── Chart.yaml              # Umbrella chart definition and child dependencies
│       ├── values.yaml            # Shared defaults
│       ├── values-dev.yaml        # Dev environment overlay
│       ├── values-prod.yaml       # Production environment overlay
│       ├── templates/
│       │   └── ingress.yaml       # Global ingress template
│       └── charts/
│           ├── postgres/
│           ├── redis/
│           ├── booking/
│           ├── mail/
│           ├── homepage/
│           ├── api-gateway/
│           └── frontend/
├── Makefile.helm                  # Helm and Kubernetes operational shortcuts
├── README.md
├── terraform/
│   ├── outputs.tf
│   └── variables.tf
└── .gitignore
```

## Core components

### 1. Argo CD application definition

The `argocd/application.yaml` file defines the desired application state:

- Application name: `sync-seat`
- Namespace: `argocd`
- Project: `sync-seat`
- Git source: `https://github.com/phanivarun05/sync-seat-gitops-eks.git`
- Target revision: `main`
- Helm chart path: `helm/syncseat`
- Destination cluster: configured Kubernetes API server endpoint
- Destination namespace: `sync-seat`

Argo CD is configured with:

- `prune: true` to remove obsolete resources
- `selfHeal: true` to reconcile drift back to Git
- `CreateNamespace=true` so the target namespace is created automatically

This is the heart of the GitOps automation: the cluster is intentionally made to converge on the state declared in Git.

### 2. Argo CD project boundary

The `argocd/project.yaml` file sets the trust boundary for the GitOps app:

- only this GitHub repo is allowed as a source
- only the `sync-seat` namespace on the target cluster is allowed as destination
- namespace-scoped resources are explicitly whitelisted, including ConfigMaps, Secrets, Services, PVCs, Deployments, Pods, and ReplicaSets

This ensures the platform is isolated and governed by a controlled deployment scope.

### 3. Helm umbrella chart

The root chart in `helm/syncseat/Chart.yaml` is an umbrella chart that brings together all platform components as dependencies:

- postgres
- redis
- booking
- mail
- homepage
- api-gateway
- frontend

Each component is defined as an independent Helm subchart. This design keeps the deployment modular and environment-friendly.

## Environment model

The repo uses value overlays so the same chart can deploy to multiple environments with different sizing and behavior.

### Shared defaults

`helm/syncseat/values.yaml` contains baseline values for the umbrella chart:

- default namespace: `sync-seat`
- infrastructure components enabled by default
- frontend and backend apps enabled by default
- ingress disabled by default unless explicitly enabled

### Development environment

`helm/syncseat/values-dev.yaml` is tuned for development or smaller Kubernetes clusters such as Kind.

Highlights:

- `global.environment.name: development`
- lower CPU and memory requests/limits
- reduced replica counts
- smaller resource footprint for Postgres and Redis
- image tags pinned to a specific commit SHA for deterministic environment behavior

This makes early validation cheaper and faster while still exercising the same deployment structure.

### Production environment

`helm/syncseat/values-prod.yaml` is tuned for production workloads.

Highlights:

- `global.environment.name: production`
- production workloads use multiple replicas
- autoscaling enabled for booking, mail, homepage, and api-gateway
- pod disruption budgets enabled for resilience
- ingress configured for hostnames such as `shop.example.com` and `api.example.com`
- static GPU/cluster performance settings are defined for the chosen platform

This environment model demonstrates a strong separation between a lighter dev deployment and a hardened production deployment.

## Deployment pipeline

The GitOps pipeline is designed as follows:

### Step 1: Source code and image delivery

The application services and frontend are built and packaged as container images, then published to the container registry. This repo assumes those images are versioned and tagged with a stable SHA or release tag.

### Step 2: Git repository becomes the source of truth

This repository contains the full desired state of the Kubernetes platform:

- Helm chart definitions
- runtime values per environment
- Argo CD application wiring
- cluster-scoped deployment policy

Any change to the repo is a deployment intent.

### Step 3: Continuous validation in GitHub Actions

The workflow in `.github/workflows/ci.yml` executes on pushes and pull requests to `main`.

It does the following:

1. checks out the repository
2. installs Helm
3. installs kubeconform
4. runs `make -f Makefile.helm helm-validate`

The validation command performs:

- `helm lint helm/syncseat`
- `helm template sync-seat helm/syncseat -f helm/syncseat/values-dev.yaml`
- kubeconform validation against the rendered manifests

This acts as a guardrail to prevent invalid Helm templates or Kubernetes manifests from merging.

### Step 4: Argo CD reconciliation

Once the branch is updated and the desired state is in Git, Argo CD watches this repository and applies the expected manifests to the target cluster.

The `syncPolicy` in `argocd/application.yaml` ensures:

- drift is corrected automatically
- resources are pruned when removed from Git
- missing namespaces are created automatically

### Step 5: Runtime behavior

The target namespace (`sync-seat`) contains the application stack, including:

- database and cache infrastructure
- web and API services
- frontend presentation layer
- ingress exposure for externally routable services

This creates a consistent, declarative runtime that can be recreated from Git at any time.

## Component breakdown

### PostgreSQL

The `postgres` chart provisions a PostgreSQL database using the official `postgres:16-alpine` image.

It includes:

- a Deployment
- a Service
- persistent volume claims
- startup, liveness, and readiness probes
- resource requests and limits

This database is the core stateful dependency for the application.

### Redis

The `redis` chart provisions a Redis cache instance with persistent storage and readiness checks. Redis supports fast data access and queue or session-like workloads used by the application services.

### Booking service

The `booking` chart deploys the booking microservice. It includes:

- container image configuration
- secret-based environment injection (`envFrom`)
- init containers that wait for PostgreSQL and Redis before startup
- secure runtime settings (non-root execution, dropped Linux capabilities)
- health probes and optional autoscaling / disruption budgets

This service is designed to be resilient and start only after dependencies are ready.

### Mail service

The `mail` chart follows the same deployment pattern as the other application services, representing the internal mail or notification component of the Sync Seat platform.

### Homepage service

The `homepage` chart deploys the homepage application and expects both PostgreSQL and Redis dependencies. It also includes secret-based config and startup dependency wiring.

### API gateway

The `api-gateway` chart provisions the public access gateway for the platform. It performs front-door routing and integrates with the rest of the application ecosystem.

### Frontend

The `frontend` chart deploys the web experience. It mounts an Nginx config from a ConfigMap and exposes the frontend service on a cluster-internal port. It is intended to serve the user-facing app and is fronted by the ingress configuration when enabled.

## Helm operations and developer workflow

The `Makefile.helm` file includes the operational lifecycle for this topology.

Common tasks:

- install infrastructure: `make -f Makefile.helm infra`
- install app services: `make -f Makefile.helm microservices`
- install frontend: `make -f Makefile.helm frontend`
- install everything: `make -f Makefile.helm all`
- view pods/services/deployments: `make -f Makefile.helm pods`, `svc`, `deploy`
- validate charts: `make -f Makefile.helm helm-lint`, `helm-template`, `k8s-validate`
- uninstall resources: `make -f Makefile.helm clean`

There are also explicit install targets for environment-level deployment:

- `make -f Makefile.helm install-sync-seat-dev`
- `make -f Makefile.helm install-sync-seat-prod`

These targets show that the team intentionally supports both development and production release paths through the same Helm chart structure.

## Ingress and external access

The umbrella chart includes an ingress template, and the environment overlays configure external host entries like:

- `shop.example.com`
- `api.example.com`

The ingress template uses standard Kubernetes networking resources to route traffic into the cluster. In production, the ingress is configured with a GCE ingress class annotation, which fits a Google Kubernetes Engine-based deployment pattern.

## Security and operational best practices

This GitOps setup incorporates several production-friendly patterns:

- declarative Git-based config rather than imperative cluster changes
- automated drift correction via Argo CD
- namespace restrictions via AppProject
- health probes for runtime correctness
- pod disruption budgets and autoscaling in production
- secret-based configuration over direct environment hardcoding
- GitHub Actions validation before deployment

These are strong foundations for a repeatable and auditable platform.

## How to use this repo

To deploy the stack from this repository:

1. Ensure the cluster is available and the Argo CD instance is installed.
2. Apply the project and application manifests from `argocd/`.
3. Ensure the target namespace exists or is created by Argo CD.
4. Commit your desired Helm changes to the repository.
5. Let Argo CD reconcile the cluster automatically.

For local or manual validation:

```bash
make -f Makefile.helm helm-validate
helm template sync-seat helm/syncseat -f helm/syncseat/values-dev.yaml
```

## Summary

This repository is the deployment backbone for the Sync Seat platform. It uses:

- Helm for packaging and templating
- Argo CD for GitOps reconciliation
- GitHub Actions for validation
- environment overlays for dev and prod separation
- modular subcharts for the microservice stack

In short, this repo turns platform configuration into a declarative, reviewable, and automatically reconciled deployment system that is easy to manage as the application grows.

## Repository relationship

This repository is the GitOps companion to the application source repository for the Sync Seat platform. Together they form a modern deployment model:

- application code repo: builds and versions the services
- GitOps repo: defines the runtime environment, cluster policy, and release behavior

This split is a common and effective pattern for production Kubernetes delivery.
