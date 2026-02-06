# Service Mesh 3 Gateway Features - Deep Dive & Experiments

## Overview

OpenShift Service Mesh 3 uses the **Kubernetes Gateway API** (a successor to Ingress) with Istio's implementation. The Gateway provides powerful Layer 7 traffic management at the cluster edge.

## Current Setup Status

Your environment already has:
- **Istio Gateway resource**: `bookinfo-gateway` (listening on port 8080)
- **Ingress Gateway Deployment**: `ingressgateway` in `istio-gateway` namespace
- **OpenShift Route**: `bookinfo` routing external traffic to the gateway
- **VirtualService**: `bookinfo` routing gateway traffic to productpage

---

## Important Gateway Features to Experiment

### 1. **Multi-Protocol Support** ⭐⭐⭐

Gateway can handle HTTP, HTTPS, TCP, and TLS traffic on different ports.

#### Experiment: Add HTTPS Support

**Current (HTTP only):**
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - '*'
```

**Enhanced (HTTP + HTTPS):**
```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway-enhanced
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  # HTTP server
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - '*'
  # HTTPS server with TLS
  - port:
      number: 8443
      name: https
      protocol: HTTPS
    hosts:
    - bookinfo.apps-crc.testing
    tls:
      mode: SIMPLE
      credentialName: bookinfo-tls-cert  # Secret with TLS cert
```

**Test Commands:**
```powershell
# Create self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=bookinfo.apps-crc.testing"

# Create Kubernetes secret
oc create secret tls bookinfo-tls-cert --cert=cert.pem --key=key.pem -n istio-gateway

# Apply enhanced gateway
oc apply -f gateway-enhanced.yaml

# Test HTTPS
curl -k https://bookinfo-istio-gateway.apps-crc.testing:8443/productpage
```

**Learning Outcomes:**
- Understand TLS termination at gateway
- Configure certificate management
- Handle mixed HTTP/HTTPS traffic

---

### 2. **Host-Based Routing** ⭐⭐⭐

Route different hostnames to different services through the same gateway.

#### Experiment: Multi-Tenant Gateway

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: multi-host-gateway
  namespace: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.apps-crc.testing      # Bookinfo app
    - productpage.apps-crc.testing   # Direct productpage access
    - reviews.apps-crc.testing       # Direct reviews access
```

**Multiple VirtualServices:**
```yaml
---
# VirtualService for bookinfo.apps-crc.testing
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bookinfo-vs
  namespace: bookinfo
spec:
  hosts:
  - bookinfo.apps-crc.testing
  gateways:
  - istio-gateway/multi-host-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: productpage
        port:
          number: 9080

---
# VirtualService for reviews.apps-crc.testing
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews-direct-vs
  namespace: bookinfo
spec:
  hosts:
  - reviews.apps-crc.testing
  gateways:
  - istio-gateway/multi-host-gateway
  http:
  - route:
    - destination:
        host: reviews
        port:
          number: 9080
        subset: v3  # Always route to v3 with red stars
```

**Test Commands:**
```powershell
# Add to hosts file
Add-Content C:\Windows\System32\drivers\etc\hosts "127.0.0.1 reviews.apps-crc.testing"

# Create OpenShift routes
oc create route passthrough reviews --service=ingressgateway --hostname=reviews.apps-crc.testing -n istio-gateway

# Test different hosts
curl http://bookinfo.apps-crc.testing/productpage
curl http://reviews.apps-crc.testing/reviews/0
```

**Learning Outcomes:**
- Multi-tenancy with single gateway
- Host-based traffic segregation
- Shared gateway infrastructure

---

### 3. **TLS Modes** ⭐⭐⭐⭐

Gateway supports multiple TLS modes for different security requirements.

#### TLS Mode Options

| Mode | Description | Use Case |
|------|-------------|----------|
| **PASSTHROUGH** | Gateway doesn't decrypt, passes TLS to backend | End-to-end encryption |
| **SIMPLE** | Gateway terminates TLS, forwards HTTP to backend | Standard HTTPS |
| **MUTUAL** | mTLS - Gateway requires client certificates | High security, API authentication |
| **AUTO_PASSTHROUGH** | SNI-based routing without terminating TLS | Multi-tenant TLS |

#### Experiment: Mutual TLS (mTLS) at Gateway

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: mtls-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8443
      name: https-mtls
      protocol: HTTPS
    hosts:
    - secure.bookinfo.apps-crc.testing
    tls:
      mode: MUTUAL
      credentialName: bookinfo-mtls-cert
      # Requires client certificate for access
```

**Setup mTLS:**
```powershell
# Generate CA
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj "/O=MyOrg/CN=my-ca" -keyout ca.key -out ca.crt

# Generate server certificate
openssl req -out server.csr -newkey rsa:2048 -nodes -keyout server.key -subj "/CN=secure.bookinfo.apps-crc.testing/O=MyOrg"
openssl x509 -req -days 365 -CA ca.crt -CAkey ca.key -set_serial 0 -in server.csr -out server.crt

# Generate client certificate
openssl req -out client.csr -newkey rsa:2048 -nodes -keyout client.key -subj "/CN=client/O=MyOrg"
openssl x509 -req -days 365 -CA ca.crt -CAkey ca.key -set_serial 1 -in client.csr -out client.crt

# Create secrets
oc create secret generic bookinfo-mtls-cert -n istio-gateway `
  --from-file=tls.key=server.key `
  --from-file=tls.crt=server.crt `
  --from-file=ca.crt=ca.crt

# Test with client cert
curl --cert client.crt --key client.key --cacert ca.crt https://secure.bookinfo.apps-crc.testing:8443/productpage
```

**Learning Outcomes:**
- Implement zero-trust network access
- Client certificate authentication
- PKI certificate management

---

### 4. **Traffic Splitting at Gateway Level** ⭐⭐⭐

Split traffic to different backends based on gateway rules.

#### Experiment: Blue/Green Deployment via Gateway

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bluegreen-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - '*'

---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bluegreen-productpage
  namespace: bookinfo
spec:
  hosts:
  - '*'
  gateways:
  - bluegreen-gateway
  http:
  # Route based on header (for testing new version)
  - match:
    - headers:
        x-version:
          exact: green
    route:
    - destination:
        host: productpage
        subset: v2  # New version
  # Default traffic to stable version
  - route:
    - destination:
        host: productpage
        subset: v1  # Stable version
```

**Test Commands:**
```powershell
# Regular users get v1
curl http://bookinfo-istio-gateway.apps-crc.testing/productpage

# Testers with header get v2
curl -H "x-version: green" http://bookinfo-istio-gateway.apps-crc.testing/productpage
```

**Learning Outcomes:**
- Blue/green deployments
- Header-based routing
- Gradual rollout strategies

---

### 5. **Cross-Namespace Gateway Sharing** ⭐⭐⭐⭐

One gateway in `istio-gateway` namespace serving apps in multiple namespaces.

#### Experiment: Shared Gateway Architecture

```yaml
# Gateway in istio-gateway namespace (already exists)
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: istio-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - '*.apps-crc.testing'  # Wildcard for multi-tenancy

---
# VirtualService in app namespace references gateway in different namespace
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: app1-vs
  namespace: bookinfo  # App namespace
spec:
  hosts:
  - app1.apps-crc.testing
  gateways:
  - istio-gateway/shared-gateway  # Reference format: namespace/name
  http:
  - route:
    - destination:
        host: productpage
        port:
          number: 9080
```

**Test Commands:**
```powershell
# Check gateway can be used by multiple namespaces
oc get virtualservices -A -o custom-columns=NAME:.metadata.name,NAMESPACE:.metadata.namespace,GATEWAYS:.spec.gateways
```

**Learning Outcomes:**
- Multi-tenant architecture
- Namespace isolation with shared infrastructure
- Gateway resource organization

---

### 6. **HTTP Redirect and Rewrite** ⭐⭐⭐

Modify requests at the gateway before forwarding.

#### Experiment: HTTP to HTTPS Redirect

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: redirect-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway
  servers:
  # HTTP server that redirects to HTTPS
  - port:
      number: 8080
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.apps-crc.testing
    tls:
      httpsRedirect: true  # Automatically redirect HTTP → HTTPS
  # HTTPS server
  - port:
      number: 8443
      name: https
      protocol: HTTPS
    hosts:
    - bookinfo.apps-crc.testing
    tls:
      mode: SIMPLE
      credentialName: bookinfo-tls-cert
```

#### Experiment: Path Rewriting

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: rewrite-vs
  namespace: bookinfo
spec:
  hosts:
  - bookinfo.apps-crc.testing
  gateways:
  - redirect-gateway
  http:
  # Rewrite /app/productpage → /productpage
  - match:
    - uri:
        prefix: /app
    rewrite:
      uri: /
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

**Test Commands:**
```powershell
# Test redirect
curl -I http://bookinfo.apps-crc.testing:8080/productpage
# Should return: 301 Moved Permanently
# Location: https://bookinfo.apps-crc.testing:8443/productpage

# Test path rewrite
curl http://bookinfo.apps-crc.testing/app/productpage
# Gets rewritten to /productpage before forwarding
```

**Learning Outcomes:**
- Security enforcement (HTTPS)
- URL normalization
- Legacy URL support

---

### 7. **Rate Limiting and Quotas** ⭐⭐⭐⭐

Protect backend services from overload.

#### Experiment: Local Rate Limiting at Gateway

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: gateway-ratelimit
  namespace: istio-gateway
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: GATEWAY
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.local_ratelimit
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          value:
            stat_prefix: http_local_rate_limiter
            token_bucket:
              max_tokens: 10
              tokens_per_fill: 10
              fill_interval: 1m
            filter_enabled:
              runtime_key: local_rate_limit_enabled
              default_value:
                numerator: 100
                denominator: HUNDRED
            filter_enforced:
              runtime_key: local_rate_limit_enforced
              default_value:
                numerator: 100
                denominator: HUNDRED
```

**Test Commands:**
```powershell
# Send rapid requests to test rate limiting
1..20 | ForEach-Object {
    curl -I http://bookinfo-istio-gateway.apps-crc.testing/productpage
    Start-Sleep -Milliseconds 100
}
# Should see 429 Too Many Requests after 10 requests
```

**Learning Outcomes:**
- DDoS protection
- API quota enforcement
- Traffic shaping

---

### 8. **Request Authentication (JWT)** ⭐⭐⭐⭐⭐

Validate JWT tokens at the gateway before allowing access.

#### Experiment: OAuth/JWT Authentication

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: istio-gateway
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "https://example.com"
    jwksUri: "https://example.com/.well-known/jwks.json"
    # OR use local JWKS for testing
    jwks: |
      {
        "keys": [
          {
            "kty": "RSA",
            "e": "AQAB",
            "n": "..."
          }
        ]
      }

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: istio-gateway
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
```

**Test Commands:**
```powershell
# Without JWT - denied
curl http://bookinfo-istio-gateway.apps-crc.testing/productpage

# With valid JWT - allowed
$token = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
curl -H "Authorization: Bearer $token" http://bookinfo-istio-gateway.apps-crc.testing/productpage
```

**Learning Outcomes:**
- Zero-trust authentication
- OAuth2/OIDC integration
- API security

---

### 9. **Custom Headers Manipulation** ⭐⭐⭐

Add, modify, or remove headers at the gateway.

#### Experiment: Header-Based Features

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: header-manipulation
  namespace: bookinfo
spec:
  hosts:
  - bookinfo.apps-crc.testing
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        prefix: /productpage
    headers:
      request:
        add:
          x-forwarded-proto: https
          x-custom-header: gateway-added
        set:
          x-request-id: "12345"
        remove:
        - x-sensitive-data
      response:
        add:
          x-served-by: istio-gateway
          cache-control: no-cache
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

**Test Commands:**
```powershell
# Check response headers
curl -v http://bookinfo-istio-gateway.apps-crc.testing/productpage 2>&1 | Select-String -Pattern "x-served-by"
```

**Learning Outcomes:**
- Security header injection
- CORS handling
- Custom routing logic

---

### 10. **Fault Injection at Gateway** ⭐⭐⭐⭐

Test application resilience by injecting failures.

#### Experiment: Chaos Engineering

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: fault-injection-vs
  namespace: bookinfo
spec:
  hosts:
  - bookinfo.apps-crc.testing
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        prefix: /productpage
    fault:
      delay:
        percentage:
          value: 10  # 10% of requests
        fixedDelay: 5s  # Delayed by 5 seconds
      abort:
        percentage:
          value: 5  # 5% of requests
        httpStatus: 503  # Return 503 error
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```

**Test Commands:**
```powershell
# Monitor response times and errors
1..50 | ForEach-Object {
    $start = Get-Date
    try {
        $response = Invoke-WebRequest -Uri "http://bookinfo-istio-gateway.apps-crc.testing/productpage" -UseBasicParsing
        $elapsed = (Get-Date) - $start
        Write-Host "Request $($_): $($response.StatusCode) - $($elapsed.TotalSeconds)s"
    } catch {
        Write-Host "Request $($_): FAILED - $($_.Exception.Message)"
    }
}
```

**Learning Outcomes:**
- Test application resilience
- Chaos engineering practices
- Error handling validation

---

## Hands-On Experiment Plan

### Week 1: Basics
1. **Day 1-2**: TLS Termination (HTTPS setup)
2. **Day 3-4**: Multi-host routing
3. **Day 5**: HTTP redirects and rewrites

### Week 2: Security
1. **Day 1-2**: Mutual TLS (mTLS)
2. **Day 3-4**: JWT authentication
3. **Day 5**: Rate limiting

### Week 3: Advanced
1. **Day 1-2**: Fault injection
2. **Day 3-4**: Header manipulation
3. **Day 5**: Multi-protocol support

### Week 4: Production Patterns
1. **Day 1-2**: Blue/green deployments
2. **Day 3-4**: Cross-namespace sharing
3. **Day 5**: Complete integration test

---

## Monitoring Gateway Behavior

### Check Gateway Configuration
```powershell
# View gateway status
oc get gateway -n bookinfo

# Check which VirtualServices use the gateway
oc get virtualservices -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.gateways}{"\n"}{end}'

# Inspect gateway pod logs
$gw = (oc get pods -n istio-gateway -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}')
oc logs -n istio-gateway $gw --tail=100

# Check Envoy configuration
oc exec -n istio-gateway $gw -- pilot-agent request GET config_dump
```

### Metrics to Watch
- **Request rate**: `istio_requests_total{destination_service_name="ingressgateway"}`
- **Error rate**: `istio_requests_total{response_code=~"5.."}`
- **Latency**: `istio_request_duration_milliseconds`

---

## Common Gateway Patterns

### Pattern 1: API Gateway
```yaml
# Aggregate multiple microservices behind single gateway
# /api/v1/users → users-service
# /api/v1/orders → orders-service
# /api/v1/products → products-service
```

### Pattern 2: Reverse Proxy
```yaml
# External clients → Gateway → Internal services
# Handle SSL termination, auth, rate limiting at edge
```

### Pattern 3: Multi-Cluster Gateway
```yaml
# Route traffic to services in multiple clusters
# Based on availability, latency, or weights
```

---

## Troubleshooting Tips

### Gateway Not Working
```powershell
# 1. Check gateway pod is running
oc get pods -n istio-gateway

# 2. Verify gateway configuration
oc get gateway <gateway-name> -n <namespace> -o yaml

# 3. Check VirtualService is attached
oc get virtualservice <vs-name> -n <namespace> -o yaml | Select-String -Pattern "gateways:"

# 4. Test from within cluster
oc run -it --rm debug --image=curlimages/curl --restart=Never -- sh
curl http://ingressgateway.istio-gateway.svc.cluster.local:80/productpage
```

### TLS Certificate Issues
```powershell
# Verify secret exists
oc get secret <cert-secret-name> -n istio-gateway

# Check certificate validity
oc get secret <cert-secret-name> -n istio-gateway -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```

---

## Next Steps

After mastering Gateway features, explore:
1. **Egress Gateway**: Control outbound traffic from mesh
2. **Service Mesh Federation**: Connect multiple meshes
3. **Ambient Mesh**: Sidecar-less architecture (SM 3 preview)
4. **Custom Envoy Filters**: Advanced traffic manipulation

## References

- [Istio Gateway Documentation](https://istio.io/latest/docs/reference/config/networking/gateway/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [OpenShift Service Mesh 3 Docs](https://docs.openshift.com/service-mesh/3.0/ossm-about.html)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/envoy/latest/)

---

## Your Current Gateway Setup

**Existing Resources:**
- Gateway: `bookinfo-gateway` (port 8080, HTTP)
- Service: `ingressgateway` (ClusterIP: 10.217.5.59)
- Route: `bookinfo` (hostname: bookinfo-istio-gateway.apps-crc.testing)
- VirtualService: `bookinfo` (routes to productpage)

**Ready to Enhance:**
Start with TLS experiment (#1) to add HTTPS support to your existing setup!
