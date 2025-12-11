# Troubleshooting: ArgoCD Gateway Health Status "Degraded"

## Problem Description

When deploying Gateway API resources (specifically `gateway.networking.k8s.io/v1` Gateway) through ArgoCD with APISIX Ingress Controller, the ArgoCD application shows a **"Degraded"** health status even though all resources are synced and functioning correctly.

### Symptoms

- ArgoCD Application Status: `Synced` but `Degraded`
- Gateway resource shows: `Accepted: False` with message "gateway proxy not found"
- Gateway resource shows: `Programmed: True`
- All HTTPRoutes are `Accepted: True` and `ResolvedRefs: True`
- Routing functionality works correctly despite the degraded status

### Root Cause

ArgoCD's default health check for Gateway resources evaluates the `Accepted` condition. In APISIX Ingress Controller, the Gateway's `Accepted` condition may show `False` with the message "gateway proxy not found" - this is a known APISIX behavior that doesn't affect actual functionality.

The correct health indicator for APISIX Gateway resources is the `Programmed` condition, not the `Accepted` condition.

## Solution

Add a custom health check for Gateway resources in ArgoCD that evaluates the `Programmed` condition instead of the `Accepted` condition.

### Step 1: Create Custom Health Check Configuration

Create a file with the custom health check Lua script:

```bash
cat > /tmp/gateway-health-patch.yaml << 'EOF'
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
        -- Check if listeners are ready
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
EOF
```

### Step 2: Apply the Custom Health Check

Patch the ArgoCD ConfigMap with the custom health check:

```bash
kubectl patch configmap argocd-cm -n argocd --patch-file /tmp/gateway-health-patch.yaml
```

### Step 3: Restart ArgoCD Application Controller

The ArgoCD application controller needs to be restarted to pick up the new configuration:

```bash
# ArgoCD uses a StatefulSet for the application controller
kubectl rollout restart statefulset argocd-application-controller -n argocd

# Wait for the rollout to complete
kubectl rollout status statefulset argocd-application-controller -n argocd --timeout=60s
```

### Step 4: Refresh the Application

Force ArgoCD to re-evaluate the application health:

```bash
kubectl patch application apisix-gateway-api -n argocd --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'
```

### Step 5: Verify the Fix

Check that the application is now healthy:

```bash
kubectl get application apisix-gateway-api -n argocd -o wide
```

Expected output:
```
NAME                 SYNC STATUS   HEALTH STATUS   REVISION
apisix-gateway-api   Synced        Healthy         <commit-hash>
```

## Verification Steps

### 1. Check Gateway Status

```bash
kubectl get gateway apisix-gateway -n apisix-argocd -o json | jq '.status.conditions'
```

You should see:
- `Programmed: True` ✅
- `Accepted: False` (this is expected with APISIX and doesn't affect functionality)

### 2. Check HTTPRoute Status

```bash
kubectl get httproute -n apisix-argocd -o json | jq '.items[] | {
  name: .metadata.name,
  accepted: (.status.parents[0].conditions[] | select(.type=="Accepted") | .status),
  resolved: (.status.parents[0].conditions[] | select(.type=="ResolvedRefs") | .status)
}'
```

All routes should show:
- `Accepted: True` ✅
- `ResolvedRefs: True` ✅

### 3. Check Application Health

```bash
kubectl get application apisix-gateway-api -n argocd -o json | jq '{
  name: .metadata.name,
  syncStatus: .status.sync.status,
  healthStatus: .status.health.status
}'
```

Should show:
```json
{
  "name": "apisix-gateway-api",
  "syncStatus": "Synced",
  "healthStatus": "Healthy"
}
```

## Understanding the Custom Health Check

The custom health check logic:

1. **Primary Check**: Looks for `Programmed: True` condition in Gateway status
2. **Secondary Check**: Verifies all listeners have `Programmed: True` condition
3. **Returns**:
   - `Healthy` if Gateway or all listeners are programmed
   - `Progressing` if waiting for Gateway to be programmed

This approach correctly evaluates APISIX Gateway health based on operational readiness rather than the `Accepted` condition.

## Why This Works

- **APISIX Behavior**: The `Accepted: False` with "gateway proxy not found" is a known APISIX message that doesn't indicate a problem
- **Correct Indicator**: The `Programmed` condition accurately reflects whether the Gateway is operational
- **Listener Status**: All listeners showing `Programmed: True` confirms the Gateway is ready to route traffic
- **HTTPRoute Status**: Routes showing `Accepted: True` confirms they're properly attached to the Gateway

## Persistence

The custom health check is stored in the `argocd-cm` ConfigMap and will persist across ArgoCD restarts. However, if you reinstall ArgoCD or reset the ConfigMap, you'll need to reapply this configuration.

### Making it Permanent

To ensure the configuration persists, you can:

1. **Add to ArgoCD Helm values** (if using Helm):
```yaml
configs:
  cm:
    resource.customizations.health.gateway.networking.k8s.io_Gateway: |
      # ... (paste the Lua script here)
```

2. **Store in Git**: Keep the patch file in your GitOps repository
3. **Document**: Add this to your cluster setup documentation

## Related Issues

- APISIX Gateway `Accepted` condition showing False is expected behavior
- This does not affect routing functionality
- HTTPRoutes can still be accepted and route traffic correctly
- The Gateway is fully operational despite the `Accepted: False` status

## References

- [ArgoCD Resource Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [Gateway API Specification](https://gateway-api.sigs.k8s.io/)
- [APISIX Ingress Controller](https://apisix.apache.org/docs/ingress-controller/getting-started/)

