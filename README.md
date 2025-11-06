# Fire GitOps Manifests

This repository contains Kubernetes manifests for the Fire application, managed via GitOps principles.

## Repository Structure

The repository is organized into separate applications for independent management, all deploying to the `fire-prod` namespace:

```
fire-manifests/
├── applications/
│   └── fire-backend/              # Fire application
│       ├── base/
│       │   ├── deployment.yaml    # Fire deployment (ghcr.io/flawless/fire)
│       │   ├── service.yaml       # ClusterIP service on port 3000
│       │   ├── configmap.yaml     # Configuration stub
│       │   ├── database.yaml      # CloudNativePG Database resource
│       │   ├── postgres-fire-user-sealed.yaml  # Database credentials (SealedSecret)
│       │   └── kustomization.yaml
│       └── overlays/prod/
│           └── kustomization.yaml # Image tag overlay
├── infrastructure/
│   ├── fire-postgres/             # PostgreSQL cluster
│   │   ├── base/
│   │   │   ├── cluster.yaml       # CloudNativePG Cluster (PostgreSQL 18)
│   │   │   └── kustomization.yaml
│   │   └── overlays/prod/
│   │       └── kustomization.yaml
│   └── fire-ingress/              # Ingress configuration
│       ├── base/
│       │   ├── ingress.yaml       # Traefik ingress for fire.last-try.org
│       │   └── kustomization.yaml
│       └── overlays/prod/
│           └── kustomization.yaml
├── .github/workflows/
│   └── update-image.yml           # Automated image update workflow
└── README.md
```

## ArgoCD Applications

You'll need to create three ArgoCD applications (or one AppOfApps):

### 1. fire-postgres
```yaml
repoURL: https://github.com/flawless/fire-manifests
path: infrastructure/fire-postgres/overlays/prod
targetRevision: master
destination:
  namespace: fire-prod
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 2. fire-backend
```yaml
repoURL: https://github.com/flawless/fire-manifests
path: applications/fire-backend/overlays/prod
targetRevision: master
destination:
  namespace: fire-prod
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### 3. fire-ingress
```yaml
repoURL: https://github.com/flawless/fire-manifests
path: infrastructure/fire-ingress/overlays/prod
targetRevision: master
destination:
  namespace: fire-prod
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**Note:** Sync waves ensure proper ordering:
- Wave -10: PostgreSQL Cluster
- Wave -8: Sealed Secret
- Wave -5: Database resource
- Wave 2: Application deployment

## How It Works

### Image Updates

1. The [fire application repository](https://github.com/flawless/fire) builds and pushes Docker images to `ghcr.io/flawless/fire`
2. After a successful build, the fire repo triggers a `repository_dispatch` event to this repo
3. The `update-image.yml` workflow receives the event and updates the image tag in `applications/fire-backend/overlays/prod/kustomization.yaml`
4. The workflow commits and pushes the change to the `master` branch
5. ArgoCD detects the change and deploys the updated image to the cluster

### Payload Structure

The `repository_dispatch` event includes the following payload:

```json
{
  "service": "fire",
  "image": "ghcr.io/flawless/fire:prod-abc123",
  "environment": "prod",
  "commit": "abc123...",
  "branch": "master"
}
```

## Building Manifests

To preview the final Kubernetes manifests for each component:

```bash
# PostgreSQL Cluster
kustomize build infrastructure/fire-postgres/overlays/prod

# Backend application
kustomize build applications/fire-backend/overlays/prod

# Ingress
kustomize build infrastructure/fire-ingress/overlays/prod
```

Or using kubectl:

```bash
kubectl kustomize infrastructure/fire-postgres/overlays/prod
kubectl kustomize applications/fire-backend/overlays/prod
kubectl kustomize infrastructure/fire-ingress/overlays/prod
```

## Required Setup

### 1. Prerequisites

- **CloudNativePG operator** installed in the cluster
- **cert-manager** with letsencrypt-prod ClusterIssuer
- **Traefik** ingress controller
- **Sealed Secrets** controller for encrypted credentials

### 2. Database Credentials

The placeholder sealed secret at `applications/fire-backend/base/postgres-fire-user-sealed.yaml` needs to be replaced with actual encrypted credentials:

```bash
# 1. Create the plain secret (do this locally, DON'T commit)
kubectl create secret generic postgres-fire-user \
  --from-literal=username=fire-user \
  --from-literal=password=YOUR_SECURE_PASSWORD \
  --from-literal=uri=postgresql://fire-user:YOUR_SECURE_PASSWORD@fire-postgres-rw:5432/fire_db \
  --namespace=fire-prod \
  --dry-run=client -o yaml > /tmp/postgres-fire-user.yaml

# 2. Seal it
kubeseal -f /tmp/postgres-fire-user.yaml \
  -w applications/fire-backend/base/postgres-fire-user-sealed.yaml \
  --controller-namespace=sealed-secrets \
  --controller-name=sealed-secrets

# 3. Commit the sealed secret
git add applications/fire-backend/base/postgres-fire-user-sealed.yaml
git commit -m "feat: add sealed database credentials"
git push

# 4. Securely delete the plain secret
rm /tmp/postgres-fire-user.yaml
```

### 3. S3 Backups (Optional)

The PostgreSQL cluster has S3 backup configuration commented out. To enable:

1. Create an S3 bucket for backups
2. Create AWS credentials secret
3. Uncomment the `backup` section in `infrastructure/fire-postgres/base/cluster.yaml`
4. Update the `destinationPath` with your bucket name

### 4. GitHub Workflow Secret

**This repository requires no secrets.** The workflow uses the default `GITHUB_TOKEN` with write access.

The fire application repository needs a `MANIFESTS_PAT` secret with `repo` scope to trigger the `repository_dispatch` event.

## Deployment

This repository is designed to work with GitOps tools like ArgoCD or Flux:

- Watch the `master` branch
- Monitor the three application paths listed above
- Auto-sync changes to the `fire-prod` namespace

## Manual Updates

To manually update the image tag:

1. Edit `applications/fire-backend/overlays/prod/kustomization.yaml`
2. Update the `newTag` value under `images:`
3. Commit and push to `master`

## Database Connection

The fire application receives the PostgreSQL connection string via the `DATABASE_URL` environment variable, which is sourced from the `postgres-fire-user` secret's `uri` field.

The CloudNativePG operator creates a service named `fire-postgres-rw` (read-write) that routes to the primary PostgreSQL instance.

## Troubleshooting

### Database won't start
- Check if CloudNativePG operator is running: `kubectl get pods -n cnpg-system`
- Check cluster status: `kubectl get cluster fire-postgres -n fire-prod`
- View cluster events: `kubectl describe cluster fire-postgres -n fire-prod`

### Application can't connect to database
- Verify secret exists: `kubectl get secret postgres-fire-user -n fire-prod`
- Check DATABASE_URL is set: `kubectl get pod -n fire-prod -l app=fire -o jsonpath='{.items[0].spec.containers[0].env}'`
- Verify database was created: `kubectl get database fire-db -n fire-prod`

### Ingress not working
- Check cert-manager issuer: `kubectl get clusterissuer letsencrypt-prod`
- Check certificate: `kubectl get certificate fire-tls -n fire-prod`
- Check ingress: `kubectl describe ingress fire -n fire-prod`

### Image not updating
- Check GitHub Actions workflow runs in both repos
- Verify `MANIFESTS_PAT` secret exists in fire repo
- Check for errors in the update-image workflow logs
