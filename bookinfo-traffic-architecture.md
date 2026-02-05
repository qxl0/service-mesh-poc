# Bookinfo Application - Complete Traffic Architecture

## Traffic Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│ EXTERNAL ACCESS                                                                 │
│                                                                                 │
│ Browser: http://bookinfo-istio-gateway.apps-crc.testing/productpage             │
│          ↓                                                                      │
│ OpenShift Router (HAProxy)                                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────┐
│ ISTIO-GATEWAY NAMESPACE                                                         │
│                                                                                 │
│ Route: bookinfo                                                                 │
│   Host: bookinfo-istio-gateway.apps-crc.testing                                 │
│   Backend: ingressgateway service                                               │
│          ↓                                                                      │
│ Service: ingressgateway                                                         │
│   Type: ClusterIP                                                               │
│   ClusterIP: 10.217.5.59                                                        │
│   Ports: 15021/TCP (health), 80/TCP (http2), 443/TCP (https)                    │
│          ↓                                                                      │
│ Pod: ingressgateway-857779bc87-485w5                                            │
│   PodIP: 10.217.0.179                                                           │
│   Container: istio-proxy (Envoy)                                                │
│   Listens on: 8080 (configured in Gateway resource)                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    ↓
                    [Gateway Resource: bookinfo-gateway]
                    [VirtualService: bookinfo]
                    Routes: /productpage → productpage:9080
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────────┐
│ BOOKINFO NAMESPACE                                                              │
│                                                                                 │
│ ┌───────────────────────────────────────────────────────────────────┐           │
│ │ Service: productpage                                              │           │
│ │   ClusterIP: 10.217.4.168                                         │           │
│ │   Port: 9080 → TargetPort: 9080                                   │           │
│ │          ↓                                                        │           │
│ │ Pod: productpage-v1-798b5fff65-8bhc9                              │           │
│ │   PodIP: 10.217.0.60                                              │           │
│ │   Containers:                                                     │           │
│ │     - productpage:9080 (Python Flask app)                         │           │
│ │     - istio-proxy (Envoy sidecar)                                 │           │
│ └───────────────────────────────────────────────────────────────────┘           │
│                          ↓                                                      │
│         ┌────────────────┼────────────────┐                                     │
│         ↓                ↓                ↓                                     │
│   ┌─────────┐      ┌─────────┐      ┌─────────┐                                 │
│   │ details │      │ reviews │      │ ratings │                                 │
│   └─────────┘      └─────────┘      └─────────┘                                 │
│         │                │                                                      │
│         ↓                ↓                                                      │
│ ┌───────────────┐  ┌─────────────────────────────────────────┐                  │
│ │ Service:      │  │ Service: reviews                        │                  │
│ │  details      │  │   ClusterIP: 10.217.5.185               │                  │
│ │ ClusterIP:    │  │   Port: 9080 → TargetPort: 9080         │                  │
│ │  10.217.4.241 │  │          ↓                              │                  │
│ │ Port: 9080    │  │ [VirtualService: reviews]               │                  │
│ │      ↓        │  │ [DestinationRule: reviews-versions]     │                  │
│ │ Pod:          │  │ Traffic Split (Canary/A/B Testing):     │                  │
│ │  details-v1   │  │   ↓          ↓          ↓               │                  │
│ │ PodIP:        │  │  v1         v2         v3               │                  │
│ │  10.217.0.230 │  │   │          │          │               │                  │
│ │ Container:    │  │ ┌─────┐  ┌─────┐  ┌─────┐               │                  │
│ │  - details:   │  │ │ Pod │  │ Pod │  │ Pod │               │                  │
│ │    9080       │  │ │ IP: │  │ IP: │  │ IP: │               │                  │
│ │  - istio-     │  │ │.234 │  │.235 │  │.236 │               │                  │
│ │    proxy      │  │ └──┬──┘  └──┬──┘  └──┬──┘               │                  │
│ └───────────────┘  │    │        │        │                  │                  │
│                    │    └────────┴────────┘                  │                  │
│                    │            ↓ (v2, v3 call ratings)      │                  │
│                    └─────────────────────────────────────────┘                  │
│                                  ↓                                              │
│                     ┌──────────────────────────┐                                │
│                     │ Service: ratings         │                                │
│                     │   ClusterIP: 10.217.5.37 │                                │
│                     │   Port: 9080             │                                │
│                     │          ↓               │                                │
│                     │ Pod: ratings-v1          │                                │
│                     │   PodIP: 10.217.0.233    │                                │
│                     │   Containers:            │                                │
│                     │     - ratings:9080       │                                │
│                     │     - istio-proxy        │                                │
│                     └──────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Complete Traffic Flow Path

### 1. External Request
```
User Browser 
  → http://bookinfo-istio-gateway.apps-crc.testing/productpage
  → OpenShift Router (HAProxy)
  → Route: bookinfo (istio-gateway namespace)
```

### 2. Ingress Gateway
```
Route Backend: ingressgateway service
  → Service: 10.217.5.59:80
  → Pod: 10.217.0.179:8080 (ingressgateway-857779bc87-485w5)
  → Envoy Proxy reads:
      - Gateway: bookinfo-gateway (listens on port 8080)
      - VirtualService: bookinfo (routes /productpage → productpage.bookinfo.svc.cluster.local:9080)
```

### 3. ProductPage Service
```
Envoy forwards to: productpage.bookinfo.svc.cluster.local:9080
  → Service: 10.217.4.168:9080
  → Pod: 10.217.0.60:9080 (productpage-v1-798b5fff65-8bhc9)
  → Sidecar: istio-proxy intercepts
  → Container: productpage:9080 (Python Flask app)
```

### 4. ProductPage Makes Backend Calls

#### 4a. ProductPage → Details
```
productpage calls: http://details:9080/details/0
  → Sidecar intercepts outbound
  → Service: 10.217.4.241:9080 (details)
  → Pod: 10.217.0.230:9080 (details-v1)
  → Sidecar intercepts inbound
  → Container: details:9080 (Ruby app)
  ← Response: Book details JSON
```

#### 4b. ProductPage → Reviews
```
productpage calls: http://reviews:9080/reviews/0
  → Sidecar intercepts outbound
  → Service: 10.217.5.185:9080 (reviews)
  → VirtualService: reviews (applies traffic splitting rules)
  → DestinationRule: reviews-versions (defines v1, v2, v3 subsets)
  → Load balanced to one of:
      - reviews-v1 (10.217.0.234:9080) → No stars, no ratings call
      - reviews-v2 (10.217.0.235:9080) → Black stars, calls ratings
      - reviews-v3 (10.217.0.236:9080) → Red stars, calls ratings
```

#### 4c. Reviews (v2/v3) → Ratings
```
reviews-v2/v3 call: http://ratings:9080/ratings/0
  → Sidecar intercepts outbound
  → Service: 10.217.5.37:9080 (ratings)
  → Pod: 10.217.0.233:9080 (ratings-v1)
  → Sidecar intercepts inbound
  → Container: ratings:9080 (Node.js app)
  ← Response: Star rating (1-5)
```

### 5. Response Aggregation
```
productpage aggregates:
  - Details from details service
  - Reviews from reviews service (which may include ratings)
  → Renders HTML page
  → Returns to istio-proxy sidecar
  → Returns to ingressgateway
  → Returns through Route to user's browser
```

## Network Components Summary

### Namespaces
- **istio-gateway**: Ingress gateway deployment
- **bookinfo**: Application services
- **istio-system**: Control plane (istiod, Kiali, Grafana, etc.)

### Services (ClusterIP:Port)
| Service | ClusterIP | Port | Namespace |
|---------|-----------|------|-----------|
| ingressgateway | 10.217.5.59 | 80, 443, 15021 | istio-gateway |
| productpage | 10.217.4.168 | 9080 | bookinfo |
| details | 10.217.4.241 | 9080 | bookinfo |
| reviews | 10.217.5.185 | 9080 | bookinfo |
| ratings | 10.217.5.37 | 9080 | bookinfo |

### Pods (PodIP:Port)
| Pod | PodIP | Container Port | App | Version |
|-----|-------|----------------|-----|---------|
| ingressgateway-857779bc87-485w5 | 10.217.0.179 | 8080 | istio-proxy | - |
| productpage-v1-798b5fff65-8bhc9 | 10.217.0.60 | 9080 | productpage | v1 |
| details-v1-6cc46ffff6-jhdp6 | 10.217.0.230 | 9080 | details | v1 |
| reviews-v1-5c7588fcc9-xmbdk | 10.217.0.234 | 9080 | reviews | v1 |
| reviews-v2-67d76b4c67-q5ns5 | 10.217.0.235 | 9080 | reviews | v2 |
| reviews-v3-7d5889954-qnmkl | 10.217.0.236 | 9080 | reviews | v3 |
| ratings-v1-666cf8dddc-rltr6 | 10.217.0.233 | 9080 | ratings | v1 |

### Istio Resources
| Type | Name | Purpose |
|------|------|---------|
| Gateway | bookinfo-gateway | Configures ingress gateway to listen on port 8080 |
| VirtualService | bookinfo | Routes /productpage to productpage service |
| VirtualService | reviews | Traffic splitting/A/B testing for reviews versions |
| DestinationRule | reviews-versions | Defines v1, v2, v3 subsets for reviews |

## Port Flow Summary

```
External:80 (Route) 
  → 10.217.5.59:80 (ingressgateway Service) 
  → 10.217.0.179:8080 (ingressgateway Pod, Envoy listening port)
  → 10.217.4.168:9080 (productpage Service)
  → 10.217.0.60:9080 (productpage Pod)
       ↓
       ├→ 10.217.4.241:9080 (details Service) → 10.217.0.230:9080 (details Pod)
       └→ 10.217.5.185:9080 (reviews Service) 
            → 10.217.0.234:9080 (reviews-v1 Pod) [no ratings call]
            → 10.217.0.235:9080 (reviews-v2 Pod) → 10.217.5.37:9080 (ratings)
            → 10.217.0.236:9080 (reviews-v3 Pod) → 10.217.5.37:9080 (ratings)
```

## Key Observations

1. **Gateway Port**: The Gateway resource is configured to listen on **port 8080** (not 80), which works because the ingressgateway pod's Envoy proxy is listening on that port.

2. **Service Mesh Traffic**: All pod-to-pod traffic is intercepted by **istio-proxy sidecars** for:
   - mTLS encryption (if PeerAuthentication enabled)
   - Traffic routing (VirtualServices)
   - Load balancing (DestinationRules)
   - Observability (metrics, traces, logs)

3. **Reviews Traffic Splitting**: The `reviews` VirtualService can route traffic to different versions:
   - **v1**: No ratings (no stars shown)
   - **v2**: Black stars (calls ratings service)
   - **v3**: Red stars (calls ratings service)

4. **Network Isolation**: 
   - Gateway in **istio-gateway** namespace
   - Apps in **bookinfo** namespace
   - Cross-namespace traffic controlled by NetworkPolicies (if configured)

5. **All app containers listen on port 9080** for consistency

## Traffic Observation Tools

- **Kiali**: Graph view shows service topology and traffic flow
- **Grafana**: Metrics dashboards for throughput, latency, errors
- **Tempo/Jaeger**: Distributed traces showing exact request path
- **Prometheus**: Raw metrics for all services

Use the commands in the main guide to access these UIs.
