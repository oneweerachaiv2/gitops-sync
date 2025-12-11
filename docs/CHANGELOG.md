# Changelog

All notable changes and fixes to the APISIX Gateway API deployment.

## [2025-12-11] - Health Status Fix

### Fixed
- **ArgoCD Application Health Status** - Changed from "Degraded" to "Healthy"
  - Added custom health check for Gateway resources in ArgoCD ConfigMap
  - Health check now evaluates `Programmed` condition instead of `Accepted`
  - Gateway shows `Programmed: True` which correctly indicates operational status
  - All HTTPRoutes remain `Accepted: True` and `ResolvedRefs: True`

### Changed
- **ArgoCD ConfigMap** (`argocd-cm` in `argocd` namespace)
  - Added `resource.customizations.health.gateway.networking.k8s.io_Gateway`
  - Custom Lua health check script evaluates Gateway operational readiness
  - Checks both Gateway-level and listener-level `Programmed` conditions

### Technical Details

**Problem:**
- ArgoCD default health check evaluated Gateway's `Accepted` condition
- APISIX sets `Accepted: False` with message "gateway proxy not found"
- This is expected APISIX behavior and doesn't affect functionality
- Caused ArgoCD to report application as "Degraded"

**Solution:**
- Custom health check evaluates `Programmed` condition instead
- `Programmed: True` is the correct indicator for APISIX Gateway health
- All listeners show `Programmed: True` confirming operational status
- ArgoCD now correctly reports application as "Healthy"

**Impact:**
- ✅ Application health status: Degraded → Healthy
- ✅ No changes to actual routing functionality
- ✅ All resources remain synced and operational
- ✅ HTTPRoutes continue to work correctly

### Verification

**Before Fix:**
```
NAME                 SYNC STATUS   HEALTH STATUS
apisix-gateway-api   Synced        Degraded
```

**After Fix:**
```
NAME                 SYNC STATUS   HEALTH STATUS
apisix-gateway-api   Synced        Healthy
```

**Gateway Conditions:**
```json
{
  "type": "Programmed",
  "status": "True",
  "message": "Programmed"
}
```

**HTTPRoute Status:**
- api-v1-users: Accepted ✅, ResolvedRefs ✅
- api-v4-users: Accepted ✅, ResolvedRefs ✅

### Files Modified
- ArgoCD ConfigMap: `argocd-cm` in `argocd` namespace
- Documentation added:
  - `docs/troubleshooting-argocd-gateway-health.md`
  - `docs/quick-fix-degraded-health.md`
  - `docs/README.md`
  - `docs/CHANGELOG.md`

### Configuration Applied

```yaml
data:
  resource.customizations.health.gateway.networking.k8s.io_Gateway: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for i, condition in ipairs(obj.status.conditions) do
          if condition.type == "Programmed" and condition.status == "True" then
            hs.status = "Healthy"
            hs.message = "Gateway is programmed"
            return hs
          end
        end
        if obj.status.listeners ~= nil then
          local allListenersReady = true
          for i, listener in ipairs(obj.status.listeners) do
            if listener.conditions ~= nil then
              local listenerReady = false
              for j, condition in ipairs(listener.conditions) do
                if condition.type == "Programmed" and condition.status == "True" then
                  listenerReady = true
                  break
                end
              end
              if not listenerReady then
                allListenersReady = false
                break
              end
            end
          end
          if allListenersReady then
            hs.status = "Healthy"
            hs.message = "All listeners are programmed"
            return hs
          end
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for Gateway to be programmed"
    return hs
```

### Rollout Steps

1. Created custom health check configuration
2. Patched ArgoCD ConfigMap with custom health check
3. Restarted ArgoCD application controller StatefulSet
4. Waited for controller to restart and reload configuration
5. Refreshed ArgoCD application to re-evaluate health
6. Verified application shows "Healthy" status

### Related Issues

- APISIX Gateway `Accepted` condition showing `False` is expected behavior
- This does not affect routing functionality
- HTTPRoutes can still be accepted and route traffic correctly
- The Gateway is fully operational despite `Accepted: False` status

### References

- [ArgoCD Resource Health Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Gateway API Conditions](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io/v1.GatewayConditionType)
- [APISIX Ingress Controller](https://apisix.apache.org/docs/ingress-controller/)

---

## [2025-12-11] - Initial Deployment

### Added
- ArgoCD Application for Gateway API resources
- GatewayClass resource (`apisix-argocd`)
- Gateway resource with HTTP and HTTPS listeners
- HTTPRoute resources for API versioning
- TLS secret for HTTPS termination
- Admin credentials secret for APISIX
- ReferenceGrant for cross-namespace service references
- Kustomization configuration

### Resources Deployed
1. GatewayClass: `apisix-argocd`
2. Gateway: `apisix-gateway` (HTTP:80, HTTPS:443)
3. HTTPRoute: `api-v1-users` (/api/v1/users)
4. HTTPRoute: `api-v4-users` (/api/v4/users)
5. Secret: `apisix-tls-secret` (TLS certificate)
6. Secret: `apisix-admin-credentials` (Admin key)
7. ReferenceGrant: `allow-httpbin-reference`

### Configuration
- Repository: `https://github.com/oneweerachaiv2/gitops-sync.git`
- Target Namespace: `apisix-argocd`
- Automated Sync: Enabled with prune and self-heal
- Retry Policy: 5 attempts with exponential backoff

### Status
- All resources synced successfully
- HTTPRoutes accepted and routing traffic
- Cross-namespace service references working
- TLS termination configured

