# Architecture Diagram

## System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Git Repository                              │
│              https://github.com/oneweerachaiv2/gitops-sync.git      │
│                                                                      │
│  gateway-api/                                                        │
│  ├── argocd-application.yaml                                        │
│  ├── gatewayclass.yaml                                              │
│  ├── gateway.yaml                                                   │
│  ├── httproute-v1.yaml                                              │
│  ├── httproute-v4.yaml                                              │
│  ├── tls-secret.yaml                                                │
│  ├── admin-credentials.yaml                                         │
│  ├── reference-grant.yaml                                           │
│  └── kustomization.yaml                                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Git Sync
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         ArgoCD (argocd namespace)                    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Application: apisix-gateway-api                           │    │
│  │  ├── Sync Status: Synced ✅                                │    │
│  │  ├── Health Status: Healthy ✅                             │    │
│  │  └── Auto-Sync: Enabled (prune + self-heal)               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  ConfigMap: argocd-cm                                      │    │
│  │  └── Custom Health Check for Gateway resources            │    │
│  │      (Evaluates "Programmed" condition)                    │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Deploys Resources
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster (apisix-argocd namespace)        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  GatewayClass: apisix-argocd                               │    │
│  │  └── Controller: apisix.apache.org/apisix-ingress-controller│   │
│  └────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              │ Referenced by                         │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Gateway: apisix-gateway                                   │    │
│  │  ├── Listeners:                                            │    │
│  │  │   ├── HTTP (port 80) - Programmed ✅                    │    │
│  │  │   └── HTTPS (port 443) - Programmed ✅                  │    │
│  │  ├── Conditions:                                           │    │
│  │  │   ├── Programmed: True ✅                               │    │
│  │  │   └── Accepted: False (expected with APISIX)           │    │
│  │  └── TLS: apisix-tls-secret                               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              │ Routes attached                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  HTTPRoute: api-v1-users                                   │    │
│  │  ├── Path: /api/v1/users                                   │    │
│  │  ├── Backend: httpbin.httpbin.svc (port 80)               │    │
│  │  ├── Header: X-API-Version: v1                             │    │
│  │  └── Status: Accepted ✅, ResolvedRefs ✅                  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  HTTPRoute: api-v4-users                                   │    │
│  │  ├── Path: /api/v4/users                                   │    │
│  │  ├── Backend: httpbin.httpbin.svc (port 80)               │    │
│  │  ├── Header: X-API-Version: v4                             │    │
│  │  └── Status: Accepted ✅, ResolvedRefs ✅                  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              │ Cross-namespace reference             │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  ReferenceGrant: allow-httpbin-reference                   │    │
│  │  └── Allows: HTTPRoute → Service (httpbin namespace)       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Secrets                                                    │    │
│  │  ├── apisix-tls-secret (TLS certificate)                   │    │
│  │  └── apisix-admin-credentials (Admin key)                  │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Routes to
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Backend Services (httpbin namespace)              │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Service: httpbin                                          │    │
│  │  └── Port: 80                                              │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

## Traffic Flow

```
Client Request
      │
      ▼
┌─────────────────┐
│  HTTP/HTTPS     │
│  Port 80/443    │
└─────────────────┘
      │
      ▼
┌─────────────────────────────────────┐
│  Gateway: apisix-gateway            │
│  ├── Listener: HTTP (80)            │
│  └── Listener: HTTPS (443)          │
└─────────────────────────────────────┘
      │
      ├─── /api/v1/users ──────────────┐
      │                                 │
      │                                 ▼
      │                    ┌────────────────────────┐
      │                    │ HTTPRoute: api-v1-users│
      │                    │ Add Header:            │
      │                    │ X-API-Version: v1      │
      │                    └────────────────────────┘
      │                                 │
      │                                 ▼
      │                    ┌────────────────────────┐
      │                    │ ReferenceGrant         │
      │                    │ (Cross-namespace OK)   │
      │                    └────────────────────────┘
      │                                 │
      │                                 ▼
      │                    ┌────────────────────────┐
      │                    │ Service: httpbin       │
      │                    │ Namespace: httpbin     │
      │                    └────────────────────────┘
      │
      └─── /api/v4/users ──────────────┐
                                        │
                                        ▼
                           ┌────────────────────────┐
                           │ HTTPRoute: api-v4-users│
                           │ Add Header:            │
                           │ X-API-Version: v4      │
                           └────────────────────────┘
                                        │
                                        ▼
                           ┌────────────────────────┐
                           │ ReferenceGrant         │
                           │ (Cross-namespace OK)   │
                           └────────────────────────┘
                                        │
                                        ▼
                           ┌────────────────────────┐
                           │ Service: httpbin       │
                           │ Namespace: httpbin     │
                           └────────────────────────┘
```

## Health Check Flow

```
┌─────────────────────────────────────────────────────────────┐
│  ArgoCD Application Controller                              │
└─────────────────────────────────────────────────────────────┘
                        │
                        │ Evaluates Health
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Custom Health Check (Lua Script)                           │
│  Location: ConfigMap argocd-cm                              │
│  Key: resource.customizations.health.gateway...Gateway      │
└─────────────────────────────────────────────────────────────┘
                        │
                        │ Checks
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Gateway Status Conditions                                  │
│  ├── Programmed: True ✅ ← Primary Check                    │
│  └── Accepted: False (ignored)                              │
└─────────────────────────────────────────────────────────────┘
                        │
                        │ Also Checks
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Listener Status Conditions                                 │
│  ├── HTTP Listener: Programmed: True ✅                     │
│  └── HTTPS Listener: Programmed: True ✅                    │
└─────────────────────────────────────────────────────────────┘
                        │
                        │ Returns
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  Health Status: Healthy ✅                                  │
│  Message: "Gateway is programmed"                           │
└─────────────────────────────────────────────────────────────┘
```

## Component Relationships

```
┌──────────────────┐
│  ArgoCD App      │
└──────────────────┘
        │
        │ manages
        ▼
┌──────────────────┐      ┌──────────────────┐
│  GatewayClass    │◄─────│  Gateway         │
└──────────────────┘ refs └──────────────────┘
                                  │
                                  │ uses
                                  ▼
                          ┌──────────────────┐
                          │  TLS Secret      │
                          └──────────────────┘
                                  │
                                  │ attached
                                  ▼
                          ┌──────────────────┐
                          │  HTTPRoute (v1)  │
                          └──────────────────┘
                                  │
                                  │ references
                                  ▼
                          ┌──────────────────┐
                          │  ReferenceGrant  │
                          └──────────────────┘
                                  │
                                  │ allows
                                  ▼
                          ┌──────────────────┐
                          │  httpbin Service │
                          └──────────────────┘
```

## Namespace Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Namespace: argocd                                          │
│  ├── Application: apisix-gateway-api                        │
│  ├── ConfigMap: argocd-cm (custom health check)            │
│  └── StatefulSet: argocd-application-controller            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Namespace: apisix-argocd                                   │
│  ├── Gateway: apisix-gateway                                │
│  ├── HTTPRoute: api-v1-users                                │
│  ├── HTTPRoute: api-v4-users                                │
│  ├── Secret: apisix-tls-secret                              │
│  ├── Secret: apisix-admin-credentials                       │
│  └── ReferenceGrant: allow-httpbin-reference                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Namespace: apisix                                          │
│  └── GatewayProxy: apisix-gateway-api-proxy                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Namespace: httpbin                                         │
│  └── Service: httpbin                                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Cluster-scoped                                             │
│  └── GatewayClass: apisix-argocd                            │
└─────────────────────────────────────────────────────────────┘
```

