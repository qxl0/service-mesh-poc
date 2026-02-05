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

## Complete Traffic Flow: Step-by-Step

### Step 1: Browser Request
```
Browser sends HTTP request:
  URL: http://bookinfo-istio-gateway.apps-crc.testing/productpage
  Destination: OpenShift Router IP (external to cluster)
```

### Step 2: OpenShift Router (HAProxy)
```
Router receives request on port 80
  Looks up hostname: bookinfo-istio-gateway.apps-crc.testing
  Finds Route: "bookinfo" (istio-gateway namespace)
  Route backend: ingressgateway Service
```

### Step 3: Route → Service
```
Route forwards to:
  Service: ingressgateway (istio-gateway namespace)
  Service ClusterIP: 10.217.5.59
  Service Port: 80
  Service TargetPort: 8080
```

### Step 4: Service → Ingress Gateway Pod
```
Service (10.217.5.59:80) forwards to:
  Pod: ingressgateway-857779bc87-485w5
  Pod IP: 10.217.0.179
  Pod Port: 8080
  Container: istio-proxy (Envoy)
```

### Step 5: Gateway Pod Receives Request
```
Pod 10.217.0.179:8080 receives HTTP request
  Container: istio-proxy (Envoy gateway)
  Envoy reads Gateway resource: bookinfo-gateway
    - Configured to listen on port 8080 ✓
    - Accepts hosts: * (all) ✓
  Envoy reads VirtualService: bookinfo
    - Attached to gateway: bookinfo-gateway ✓
    - Matches URI: /productpage ✓
    - Destination: productpage:9080
```

### Step 6: Gateway → ProductPage Service
```
Gateway Pod (10.217.0.179) makes outbound call to:
  DNS: productpage.bookinfo.svc.cluster.local
  Resolves to Service: productpage
  Service ClusterIP: 10.217.4.168
  Service Port: 9080
```

### Step 7: ProductPage Service → Pod
```
Service (10.217.4.168:9080) forwards to:
  Pod: productpage-v1-798b5fff65-8bhc9
  Pod IP: 10.217.0.60
  Pod Port: 9080
```

### Step 8: ProductPage Pod Receives Request
```
Pod 10.217.0.60:9080 receives request
  Container 1: istio-proxy (sidecar)
    - Intercepts inbound traffic on port 15006 (Envoy internal)
    - Applies security policies (mTLS if enabled)
    - Forwards to localhost:9080
  Container 2: productpage
    - Listening on port 9080
    - Python Flask application
    - Receives HTTP GET /productpage
```

### Step 9: ProductPage Makes Backend Calls

#### 9a. ProductPage → Details
```
productpage container makes call:
  URL: http://details:9080/details/0
  Outbound intercepted by sidecar (10.217.0.60)
    ↓
  DNS: details.bookinfo.svc.cluster.local
  Service: 10.217.4.241:9080
    ↓
  Pod: details-v1-6cc46ffff6-jhdp6
  Pod IP: 10.217.0.230:9080
    - istio-proxy intercepts inbound
    - details container (Ruby) receives on port 9080
    - Returns book details JSON
```

#### 9b. ProductPage → Reviews
```
productpage container makes call:
  URL: http://reviews:9080/reviews/0
  Outbound intercepted by sidecar (10.217.0.60)
    ↓
  DNS: reviews.bookinfo.svc.cluster.local
  Service: 10.217.5.185:9080
    ↓
  VirtualService "reviews" applies routing rules
  Load balances to one of:
    - Pod: reviews-v1 (10.217.0.234:9080) - No stars
    - Pod: reviews-v2 (10.217.0.235:9080) - Black stars
    - Pod: reviews-v3 (10.217.0.236:9080) - Red stars
```

#### 9c. Reviews (v2/v3) → Ratings
```
reviews-v2/v3 container makes call:
  URL: http://ratings:9080/ratings/0
  Outbound intercepted by sidecar (10.217.0.235)
    ↓
  DNS: ratings.bookinfo.svc.cluster.local
  Service: 10.217.5.37:9080
    ↓
  Pod: ratings-v1-666cf8dddc-rltr6
  Pod IP: 10.217.0.233:9080
    - istio-proxy intercepts inbound
    - ratings container (Node.js) receives on port 9080
    - Returns star rating (1-5)
```

### Step 10: Response Aggregation
```
productpage container (10.217.0.60:9080):
  - Aggregates responses from details, reviews (with or without ratings)
  - Renders HTML page
  - Returns HTTP 200 with HTML content
    ↓
  istio-proxy sidecar (outbound)
    ↓
  Gateway Pod (10.217.0.179:8080)
  istio-proxy Envoy
    ↓
  Service (10.217.5.59:80)
    ↓
  Route: bookinfo
    ↓
  OpenShift Router
    ↓
  Browser displays page
```

## Summary Table: Pods and Ports

| Step | Component | Pod IP | Port | What Happens |
|------|-----------|--------|------|--------------|
| 1 | Browser | - | 80 | Sends HTTP request |
| 2 | OpenShift Router | - | 80 | Routes by hostname |
| 3 | ingressgateway Service | 10.217.5.59 | 80→8080 | Forwards to pod |
| 4 | **ingressgateway Pod** | **10.217.0.179** | **8080** | **Envoy receives request** |
| 5 | Gateway/VS resources | - | - | Routing rules applied |
| 6 | productpage Service | 10.217.4.168 | 9080 | Forwards to pod |
| 7 | **productpage Pod** | **10.217.0.60** | **9080** | **Flask app receives** |
| 8 | details Service | 10.217.4.241 | 9080 | Forwards to pod |
| 9 | **details Pod** | **10.217.0.230** | **9080** | **Ruby app returns JSON** |
| 10 | reviews Service | 10.217.5.185 | 9080 | Load balances to v1/v2/v3 |
| 11 | **reviews-v1 Pod** | **10.217.0.234** | **9080** | **Java app (no stars)** |
| 11 | **reviews-v2 Pod** | **10.217.0.235** | **9080** | **Java app (black stars)** |
| 11 | **reviews-v3 Pod** | **10.217.0.236** | **9080** | **Java app (red stars)** |
| 12 | ratings Service | 10.217.5.37 | 9080 | Forwards to pod |
| 13 | **ratings Pod** | **10.217.0.233** | **9080** | **Node.js returns rating** |

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

6. **Sidecar Interception**: The istio-proxy sidecar uses iptables to intercept:
   - **Inbound**: Traffic to port 9080 redirected to Envoy on port 15006
   - **Outbound**: Calls to other services redirected through Envoy for routing/observability

7. **Transparent to Applications**: Apps use simple service names (http://details:9080) - Istio handles DNS, load balancing, and routing

## Traffic Observation Tools

- **Kiali**: Graph view shows service topology and traffic flow
- **Grafana**: Metrics dashboards for throughput, latency, errors
- **Tempo/Jaeger**: Distributed traces showing exact request path
- **Prometheus**: Raw metrics for all services

Use the commands in the main guide to access these UIs.
