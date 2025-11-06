# Fire GitOps Manifests

Kubernetes manifests for the Fire application, managed via GitOps.

## Structure

Three independent applications deploying to `fire-prod` namespace:

- `infrastructure/fire-postgres/overlays/prod` - PostgreSQL 18 cluster (CloudNativePG)
- `applications/fire-backend/overlays/prod` - Fire application deployment
- `infrastructure/fire-ingress/overlays/prod` - Ingress for fire.last-try.org

**Sync wave ordering:**
- Wave -15: Database credentials (SealedSecret)
- Wave -10: PostgreSQL Cluster
- Wave -5: Database resource
- Wave 2: Application deployment

## Prerequisites

- CloudNativePG operator
- cert-manager with letsencrypt-prod ClusterIssuer
- Traefik ingress controller
- Sealed Secrets controller

## Setup

### 1. Seal Database Credentials

The secret at `applications/fire-backend/base/postgres-fire-user-sealed.yaml` is already sealed and committed.

To rotate credentials, create a new secret with fields `username`, `password`, `uri` and seal it using `kubeseal` with your cluster's public key.

### 2. Create ArgoCD Applications

Create three ArgoCD apps pointing to the paths above, all targeting namespace `fire-prod` with automated sync enabled.

### 3. Optional: Enable S3 Backups

Uncomment the `backup` section in `infrastructure/fire-postgres/base/cluster.yaml` and configure your S3 bucket.

## Image Updates

The `.github/workflows/update-image.yml` workflow listens for `repository_dispatch` events from the fire app repo. When a new image is built, it updates `applications/fire-backend/overlays/prod/kustomization.yaml` and commits.

**Required:** The fire repo needs a `MANIFESTS_PAT` secret with `repo` scope.

## Database Connection

The fire app receives PostgreSQL connection via `DATABASE_URL` environment variable, sourced from the `postgres-fire-user` secret. CloudNativePG creates the `fire-postgres-rw` service for connections.
