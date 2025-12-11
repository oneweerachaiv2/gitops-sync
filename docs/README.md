# Documentation

This directory contains documentation for the APISIX Gateway API deployment with ArgoCD.

## Quick Links

### Troubleshooting

- **[Quick Fix: Degraded Health Status](quick-fix-degraded-health.md)** - Fast solution for ArgoCD showing Gateway as "Degraded"
- **[Detailed Troubleshooting Guide](troubleshooting-argocd-gateway-health.md)** - Complete explanation and step-by-step fix

## Common Issues

### ArgoCD Application Shows "Degraded" Health Status

**Symptom:** ArgoCD application is `Synced` but shows `Degraded` health status

**Quick Fix:** See [quick-fix-degraded-health.md](quick-fix-degraded-health.md)

**Detailed Guide:** See [troubleshooting-argocd-gateway-health.md](troubleshooting-argocd-gateway-health.md)

**Root Cause:** ArgoCD's default health check evaluates the Gateway's `Accepted` condition, which APISIX sets to `False` (expected behavior). The actual health indicator is the `Programmed` condition.

**Solution:** Add custom health check to ArgoCD ConfigMap that evaluates `Programmed` instead of `Accepted`.

## Architecture Overview

### Components

1. **ArgoCD Application** (`apisix-gateway-api`)
   - Manages Gateway API resources
   - Syncs from Git repository
   - Automated sync with self-heal enabled

2. **Gateway API Resources**
   - **GatewayClass** (`apisix-argocd`) - Defines the gateway controller
   - **Gateway** (`apisix-gateway`) - HTTP/HTTPS listeners on ports 80/443
   - **HTTPRoutes** - Route definitions for API endpoints
   - **ReferenceGrant** - Allows cross-namespace service references

3. **Supporting Resources**
   - **TLS Secret** - Certificate for HTTPS listener
   - **Admin Credentials** - APISIX admin API authentication
   - **GatewayProxy** - APISIX-specific CRD for control plane connection

### Resource Relationships

```
ArgoCD Application (apisix-gateway-api)
  └── Manages
      ├── GatewayClass (apisix-argocd)
      │   └── Referenced by Gateway
      ├── Gateway (apisix-gateway)
      │   ├── Uses TLS Secret (apisix-tls-secret)
      │   └── Referenced by HTTPRoutes
      ├── HTTPRoute (api-v1-users)
      │   └── Routes to httpbin service
      ├── HTTPRoute (api-v4-users)
      │   └── Routes to httpbin service
      ├── ReferenceGrant (allow-httpbin-reference)
      │   └── Allows cross-namespace references
      ├── Secret (apisix-tls-secret)
      └── Secret (apisix-admin-credentials)
```

## Health Check Logic

### Default ArgoCD Behavior

ArgoCD evaluates Gateway health by checking:
- `Accepted` condition status

### APISIX Behavior

APISIX Gateway shows:
- `Accepted: False` with message "gateway proxy not found" (expected)
- `Programmed: True` (actual operational status)

### Custom Health Check

Our custom health check evaluates:
1. Primary: `Programmed: True` condition
2. Secondary: All listeners have `Programmed: True`

Result: ArgoCD correctly reports `Healthy` when Gateway is operational.

## Deployment Flow

1. **Git Commit** - Changes pushed to repository
2. **ArgoCD Sync** - Detects changes and syncs
3. **Resource Creation** - Kubernetes resources created/updated
4. **APISIX Controller** - Processes Gateway API resources
5. **Health Evaluation** - Custom health check evaluates status
6. **Status Update** - ArgoCD shows `Healthy` status

## Verification Commands

### Check Application Status
```bash
kubectl get application apisix-gateway-api -n argocd -o wide
```

### Check Gateway Status
```bash
kubectl get gateway apisix-gateway -n apisix-argocd
```

### Check HTTPRoute Status
```bash
kubectl get httproute -n apisix-argocd
```

### Check All Resources
```bash
kubectl get application apisix-gateway-api -n argocd -o json | \
  jq '.status.resources[] | {name: .name, kind: .kind, status: .status}'
```

### Verify Gateway Conditions
```bash
kubectl get gateway apisix-gateway -n apisix-argocd -o json | \
  jq '.status.conditions[] | {type: .type, status: .status, message: .message}'
```

### Verify HTTPRoute Conditions
```bash
kubectl get httproute -n apisix-argocd -o json | \
  jq '.items[] | {
    name: .metadata.name,
    accepted: (.status.parents[0].conditions[] | select(.type=="Accepted") | .status),
    resolved: (.status.parents[0].conditions[] | select(.type=="ResolvedRefs") | .status)
  }'
```

## Configuration Files

### ArgoCD Application
- **Location:** `gateway-api/argocd-application.yaml`
- **Purpose:** Defines the ArgoCD application
- **Key Settings:**
  - Repository: `https://github.com/oneweerachaiv2/gitops-sync.git`
  - Path: `gateway-api`
  - Namespace: `apisix-argocd`
  - Automated sync with prune and self-heal

### Gateway Resources
- **GatewayClass:** `gateway-api/gatewayclass.yaml`
- **Gateway:** `gateway-api/gateway.yaml`
- **HTTPRoutes:** `gateway-api/httproute-v1.yaml`, `gateway-api/httproute-v4.yaml`
- **Secrets:** `gateway-api/tls-secret.yaml`, `gateway-api/admin-credentials.yaml`
- **ReferenceGrant:** `gateway-api/reference-grant.yaml`

### Kustomization
- **Location:** `gateway-api/kustomization.yaml`
- **Purpose:** Defines resources to deploy and common labels

## Maintenance

### Updating TLS Certificate

1. Generate new certificate:
   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout tls.key -out tls.crt \
     -subj "/CN=*.example.com/O=APISIX" \
     -addext "subjectAltName=DNS:*.example.com,DNS:localhost"
   ```

2. Create secret:
   ```bash
   kubectl create secret tls apisix-tls-secret \
     --cert=tls.crt --key=tls.key \
     -n apisix-argocd --dry-run=client -o yaml > gateway-api/tls-secret.yaml
   ```

3. Commit and push to Git

### Adding New Routes

1. Create HTTPRoute YAML in `gateway-api/`
2. Add to `kustomization.yaml` resources list
3. Commit and push to Git
4. ArgoCD will automatically sync

## Support

For issues or questions:
1. Check troubleshooting guides in this directory
2. Verify Gateway and HTTPRoute conditions
3. Check ArgoCD application logs
4. Review APISIX ingress controller logs

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [APISIX Ingress Controller](https://apisix.apache.org/docs/ingress-controller/)

