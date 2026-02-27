# GitOps Setup for Omnitools

This document describes the GitOps configuration for deploying Omnitools using ArgoCD.

## Architecture

```
┌─────────────────────┐
│   omni-tools repo   │
│  (Application Code) │
└──────────┬──────────┘
           │
           │ Push to main
           ▼
    ┌──────────────┐
    │ GitHub       │
    │ Actions      │
    │ CI/CD        │
    └──────┬───────┘
           │
           ├──► Build & Test
           ├──► Build Docker Image
           │    (arnaduga/omnitools:main-<sha>)
           └──► Push to DockerHub
                │
                ▼
         ┌──────────────────┐
         │ Update Manifest  │
         │ in omnitools-argo│
         └────────┬─────────┘
                  │
                  ▼
           ┌──────────────┐
           │ ArgoCD       │
           │ Auto-Sync    │
           └──────┬───────┘
                  │
                  ▼
           ┌──────────────┐
           │ K8s Cluster  │
           │ (omnitools   │
           │  namespace)  │
           └──────────────┘
```

## Repositories

- **Application Code**: https://github.com/arnaduga/omni-tools
  - Contains the React application source code
  - Dockerfile for building the container image
  - GitHub Actions workflows for CI/CD

- **Kubernetes Manifests**: https://github.com/arnaduga/omnitools-argo
  - Contains Kubernetes deployment manifests
  - Monitored by ArgoCD for automatic deployment
  - Auto-updated by GitHub Actions from omni-tools repo

## Image Tagging Strategy

Docker images are tagged with the following patterns:

- `arnaduga/omnitools:latest` - Latest build from main branch
- `arnaduga/omnitools:main-<short-sha>` - Specific commit from main branch
- `arnaduga/omnitools:v1.2.3` - Semantic version (from git tags)

## Workflow

1. **Developer pushes code** to `arnaduga/omni-tools` main branch
2. **GitHub Actions** automatically:
   - Runs tests (unit + e2e)
   - Builds the application
   - Creates multi-platform Docker image (amd64/arm64)
   - Pushes image to DockerHub
   - Updates the manifest in `arnaduga/omnitools-argo`
3. **ArgoCD** detects the manifest change and automatically deploys to the cluster

## GitHub Secrets Required

Configure these secrets in the **omni-tools** repository:

### In `arnaduga/omni-tools` repository:

```bash
DOCKERHUB_USERNAME        # Your DockerHub username
DOCKERHUB_TOKEN           # DockerHub access token (not password!)
MANIFEST_REPO_TOKEN       # GitHub PAT with repo access to omnitools-argo
```

### Creating the secrets:

1. **DOCKERHUB_TOKEN**:
   - Go to https://hub.docker.com/settings/security
   - Create a new access token with Read & Write permissions

2. **MANIFEST_REPO_TOKEN**:
   - Go to https://github.com/settings/tokens
   - Create a Fine-grained personal access token
   - Repository access: Only select `arnaduga/omnitools-argo`
   - Permissions: Contents (Read and Write)

3. **Add secrets to GitHub**:
   - Go to https://github.com/arnaduga/omni-tools/settings/secrets/actions
   - Click "New repository secret"
   - Add each secret

## Manual Operations

### Force ArgoCD Sync
```bash
kubectl patch application omnitools -n argocd \
  --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"main"}}}'
```

### Check Deployment Status
```bash
# Check ArgoCD application
kubectl get application omnitools -n argocd

# Check deployed pods
kubectl get pods -n omnitools

# Check current image version
kubectl get deployment omnitools-deployment -n omnitools -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### Rollback to Previous Version
```bash
# View deployment history
kubectl rollout history deployment/omnitools-deployment -n omnitools

# Rollback to previous version
kubectl rollout undo deployment/omnitools-deployment -n omnitools
```

## Monitoring

### View Application Logs
```bash
kubectl logs -n omnitools -l app=omnitools -f
```

### Check ArgoCD Application Status
```bash
kubectl describe application omnitools -n argocd
```

### Access ArgoCD UI
Get the admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Port-forward to access UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then access: https://localhost:8080

## Troubleshooting

### Image not updating
1. Check if GitHub Actions workflow completed successfully
2. Verify the manifest was updated in omnitools-argo repo
3. Check ArgoCD sync status: `kubectl get application omnitools -n argocd`
4. Manually trigger sync if needed

### Build failures
1. Check GitHub Actions logs in omni-tools repo
2. Verify all required secrets are configured
3. Check DockerHub for image availability

### Deployment issues
1. Check pod status: `kubectl get pods -n omnitools`
2. View pod logs: `kubectl logs -n omnitools <pod-name>`
3. Describe pod for events: `kubectl describe pod -n omnitools <pod-name>`
