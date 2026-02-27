# OmniTools Deployment

Self-hosted collection of powerful web-based tools for everyday tasks. All files are processed entirely on the client side - nothing ever leaves your device.

## Features

- Image/Video/Audio manipulation
- PDF operations
- Text and list processing
- Date and time calculations
- Math tools
- Data tools (JSON, CSV, XML)
- No ads, no tracking
- Fast, accessible utilities

## Docker Image

- **Image**: `arnaduga/omnitools:latest`
- **Source**: Docker Hub (auto-built via GitHub Actions)
- **Port**: 80 (HTTP)
- **License**: MIT

## GitOps Setup

This repository is managed by ArgoCD for automatic deployment. See [GITOPS.md](GITOPS.md) for details.

- **Application Repo**: https://github.com/arnaduga/omni-tools
- **Manifests Repo**: https://github.com/arnaduga/omnitools-argo (this repo)
- **Deployment**: Automatic via ArgoCD when manifests change

## Deployment Instructions

### 1. Deploy the application

```bash
kubectl apply -f omnitool.yaml
```

This will create:
- Namespace: `omnitool`
- Deployment: `omnitool-deployment` (1 replica)
- Service: `omnitool-service` (ClusterIP on port 80)

### 2. Update Cloudflare Tunnel

The Cloudflare tunnel configuration has been updated to include the ingress rule for `omnitools.zgurl.cc`.

Apply the updated configuration:

```bash
kubectl apply -f cloudflare/cloudflare-tunnel.yaml
```

### 3. Restart Cloudflare tunnel

Restart the tunnel deployment to pick up the new configuration:

```bash
kubectl rollout restart deployment/cloudflared -n traefik
```

### 4. Create DNS record

Create a CNAME record in Cloudflare DNS to route traffic through the tunnel:

```bash
cloudflared tunnel route dns 3d1a2cbb-6ab3-4c52-a18a-df1a7557aa94 omnitools.zgurl.fr
```

Alternatively, create the CNAME manually in the Cloudflare dashboard:
- **Type**: CNAME
- **Name**: `omnitools.zgurl.fr`
- **Target**: `3d1a2cbb-6ab3-4c52-a18a-df1a7557aa94.cfargotunnel.com`
- **Proxy status**: Proxied

## Verify Deployment

Check the deployment status:

```bash
# Check pods
kubectl get pods -n omnitool

# Check service
kubectl get svc -n omnitool

# Check logs
kubectl logs -n omnitool -l app=omnitool

# Check tunnel logs
kubectl logs -n traefik -l app=cloudflared
```

## Access

Once deployed and DNS is propagated, access the application at:

**https://omnitools.zgurl.fr**

## Security Configuration

The deployment follows the cluster's security best practices:

- Non-root user (UID/GID 101)
- Seccomp profile enabled
- No privilege escalation
- All capabilities dropped
- Pod Security Standards: baseline
- Resource limits configured
- Health checks (liveness and readiness probes)

## Troubleshooting

### Pods not starting

```bash
kubectl describe pod -n omnitool -l app=omnitool
```

### Service not accessible

```bash
# Test internal connectivity
kubectl run -it --rm debug --image=busybox --restart=Never -- wget -O- http://omnitool-service.omnitool.svc.cluster.local

# Check tunnel configuration
kubectl get cm cloudflared-config -n traefik -o yaml
```

### DNS not resolving

```bash
# Check DNS records
nslookup omnitools.zgurl.fr

# Verify tunnel route
cloudflared tunnel route dns 3d1a2cbb-6ab3-4c52-a18a-df1a7557aa94
```

## Resource Usage

- **CPU**: 50m (request), 200m (limit)
- **Memory**: 64Mi (request), 128Mi (limit)
- **Storage**: None required (stateless application)

## Uninstall

To remove the application:

```bash
# Delete the deployment
kubectl delete -f omnitool.yaml

# Remove the ingress rule from cloudflare-tunnel.yaml
# and reapply the configuration

# Delete DNS record
cloudflared tunnel route dns delete 3d1a2cbb-6ab3-4c52-a18a-df1a7557aa94 omnitools.zgurl.fr
```
