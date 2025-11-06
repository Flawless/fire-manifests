# Fire GitOps Manifests

This repository contains Kubernetes manifests for the Fire application, managed via GitOps principles.

## Repository Structure

```
fire-manifests/
├── applications/
│   └── fire/
│       ├── base/                   # Base Kubernetes resources
│       │   ├── deployment.yaml     # Fire deployment (ghcr.io/flawless/fire)
│       │   ├── service.yaml        # ClusterIP service on port 3000
│       │   ├── configmap.yaml      # Configuration stub
│       │   └── kustomization.yaml  # Base kustomization
│       └── overlays/
│           └── prod/               # Production environment overlay
│               └── kustomization.yaml  # Prod-specific config with image tag
├── .github/
│   └── workflows/
│       └── update-image.yml        # Automated image update workflow
└── README.md
```

## How It Works

### Image Updates

1. The [fire application repository](https://github.com/flawless/fire) builds and pushes Docker images to `ghcr.io/flawless/fire`
2. After a successful build, the fire repo triggers a `repository_dispatch` event to this repo
3. The `update-image.yml` workflow receives the event and updates the image tag in `applications/fire/overlays/prod/kustomization.yaml`
4. The workflow commits and pushes the change to the `master` branch
5. ArgoCD (or your GitOps tool) detects the change and deploys the updated image to the cluster

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

To preview the final Kubernetes manifests for production:

```bash
kustomize build applications/fire/overlays/prod
```

Or using kubectl:

```bash
kubectl kustomize applications/fire/overlays/prod
```

## Required Secrets

The `update-image.yml` workflow requires GitHub App credentials for write access:

- `APP_ID`: GitHub App ID
- `APP_PRIVATE_KEY`: GitHub App private key

Alternatively, you can replace the app token step with a `REPO_PAT` personal access token.

## Deployment

This repository is designed to work with GitOps tools like ArgoCD or Flux. Configure your GitOps tool to:

- Watch the `master` branch
- Monitor `applications/fire/overlays/prod/`
- Auto-sync changes to the `fire-prod` namespace

## Manual Updates

To manually update the image tag:

1. Edit `applications/fire/overlays/prod/kustomization.yaml`
2. Update the `newTag` value under `images:`
3. Commit and push to `master`
