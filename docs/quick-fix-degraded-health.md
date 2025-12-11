# Quick Fix: ArgoCD Gateway "Degraded" Health Status

## TL;DR

ArgoCD shows Gateway as "Degraded" because it checks the `Accepted` condition. APISIX sets this to `False` (expected behavior). Fix: Add custom health check that evaluates `Programmed` condition instead.

## One-Command Fix

```bash
# 1. Apply custom health check
kubectl patch configmap argocd-cm -n argocd --type merge -p '
data:
  resource.customizations.health.gateway.networking.k8s.io_Gateway: |
    hs = {}
    if obj.status ~= nil and obj.status.conditions ~= nil then
      for i, condition in ipairs(obj.status.conditions) do
        if condition.type == "Programmed" and condition.status == "True" then
          hs.status = "Healthy"
          hs.message = "Gateway is programmed"
          return hs
        end
      end
    end
    hs.status = "Progressing"
    hs.message = "Waiting for Gateway to be programmed"
    return hs
'

# 2. Restart ArgoCD controller
kubectl rollout restart statefulset argocd-application-controller -n argocd

# 3. Wait for restart
kubectl rollout status statefulset argocd-application-controller -n argocd --timeout=60s

# 4. Refresh application (replace with your app name)
kubectl patch application apisix-gateway-api -n argocd --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# 5. Verify
kubectl get application apisix-gateway-api -n argocd
```

## Expected Result

**Before:**
```
NAME                 SYNC STATUS   HEALTH STATUS
apisix-gateway-api   Synced        Degraded
```

**After:**
```
NAME                 SYNC STATUS   HEALTH STATUS
apisix-gateway-api   Synced        Healthy
```

## What This Does

1. **Adds custom health check** to ArgoCD ConfigMap
2. **Changes evaluation** from `Accepted` to `Programmed` condition
3. **Restarts controller** to load new configuration
4. **Refreshes app** to re-evaluate health
5. **Result**: Application shows as `Healthy`

## Verification

```bash
# Check Gateway is programmed
kubectl get gateway apisix-gateway -n apisix-argocd -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
# Should output: True

# Check application health
kubectl get application apisix-gateway-api -n argocd -o jsonpath='{.status.health.status}'
# Should output: Healthy
```

## Why This Works

- APISIX Gateway shows `Accepted: False` (expected, doesn't affect functionality)
- APISIX Gateway shows `Programmed: True` (actual health indicator)
- Custom health check uses `Programmed` instead of `Accepted`
- ArgoCD now correctly reports `Healthy` status

## Troubleshooting

**If still showing Degraded:**

1. Wait 30 seconds for controller to restart
2. Check controller is running:
   ```bash
   kubectl get pods -n argocd | grep application-controller
   ```
3. Force refresh again:
   ```bash
   argocd app get apisix-gateway-api --refresh
   ```

**If ConfigMap patch fails:**

Use the detailed method in `troubleshooting-argocd-gateway-health.md`

## Rollback

To remove the custom health check:

```bash
kubectl patch configmap argocd-cm -n argocd --type json \
  -p '[{"op": "remove", "path": "/data/resource.customizations.health.gateway.networking.k8s.io_Gateway"}]'

kubectl rollout restart statefulset argocd-application-controller -n argocd
```

## See Also

- Full documentation: `docs/troubleshooting-argocd-gateway-health.md`
- ArgoCD Health Checks: https://argo-cd.readthedocs.io/en/stable/operator-manual/health/

