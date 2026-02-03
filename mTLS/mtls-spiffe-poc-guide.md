# PoC Guide: mTLS + SPIFFE with OpenShift Service Mesh 3 (Bookinfo)

This guide demonstrates how to enable **STRICT mTLS**, verify **SPIFFE
workload identities**, and enforce **identity-based authorization**
using Istio AuthorizationPolicy on OpenShift Service Mesh 3 with the
Bookinfo sample.

------------------------------------------------------------------------

## Prerequisites

-   OpenShift Local (CRC)
-   Red Hat Service Mesh 3 installed
-   Bookinfo application deployed
-   Logged in as `kubeadmin`
-   `oc` CLI available

------------------------------------------------------------------------

## 1. Verify Bookinfo and Sidecars

``` bash
oc get pods -n bookinfo
```

All Bookinfo pods should show `2/2` containers (app + istio-proxy).

------------------------------------------------------------------------

## 2. Enable STRICT mTLS in Bookinfo

``` bash
cat <<EOF | oc apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

Verify:

``` bash
oc get peerauthentication -n bookinfo
```

Test baseline:

``` bash
REVIEWS=$(oc get pod -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}')
oc exec -n bookinfo $REVIEWS -c reviews --   curl -sS -o /dev/null -w "HTTP=%{http_code}
" http://details:9080/details/0
```

Expected:

    HTTP=200

------------------------------------------------------------------------

## 3. Verify SPIFFE Identity

From productpage sidecar:

```powershell 
PP=$(oc get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
oc exec -n bookinfo $PP -c istio-proxy --   curl -s http://localhost:15000/certs | grep spiffe
```

Example:

    spiffe://cluster.local/ns/bookinfo/sa/bookinfo-productpage

![authorization-policy](mTLS-images/2026-01-27_20-18-38.PNG)
------------------------------------------------------------------------

## 4. Create AuthorizationPolicy (Identity Based)

⚠️ SM3 quirk: do NOT include `spiffe://` prefix in principals.

```powershell 
cat <<EOF | oc apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: details-allow-only-productpage
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: details
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/bookinfo/sa/bookinfo-productpage"
EOF
```
![authorization-policy](mTLS-images/2026-01-27_20-11-13.PNG)
------------------------------------------------------------------------

## 5. Create Test Pod Using Productpage ServiceAccount

```powershell 
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: spiffe-test-productpage
  namespace: bookinfo
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  serviceAccountName: bookinfo-productpage
  containers:
  - name: sleep
    image: curlimages/curl:8.6.0
    command: ["sleep","365000"]
EOF
```
![authorization-policy](mTLS-images/2026-01-27_20-27-12.PNG)
------------------------------------------------------------------------
Wait:

```powershell 
oc wait -n bookinfo --for=condition=Ready pod/spiffe-test-productpage
```

------------------------------------------------------------------------

## 6. Validate Policy

### reviews → details (DENIED)

```powershell 
REVIEWS=$(oc get pod -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}')
oc exec -n bookinfo $REVIEWS -c reviews --   curl -sS -o /dev/null -w "HTTP=%{http_code}
" http://details:9080/details/0
```

Expected:

    HTTP=403

![reviews->details](mTLS-images/2026-01-27_20-00-33.PNG)

### productpage identity → details (ALLOWED)

```powershell 
oc exec -n bookinfo spiffe-test-productpage -c sleep --   curl -sS -o /dev/null -w "HTTP=%{http_code}
" http://details:9080/details/0
```

Expected:

    HTTP=200
![reviews->details](mTLS-images/2026-01-27_20-02-50.PNG)
------------------------------------------------------------------------

## 7. Visualize Traffic in Kiali

### 7.1 Open Kiali

```powershell 
oc get route -n istio-system kiali
```

Open the URL in browser and log in.

### 7.2 Generate Traffic

``` powershell 
# generate allowed traffic
for i in {1..10}; do
  oc exec -n bookinfo spiffe-test-productpage -c sleep --     curl -s http://details:9080/details/0 > /dev/null
done

# generate denied traffic
for i in {1..10}; do
  oc exec -n bookinfo $REVIEWS -c reviews --     curl -s http://details:9080/details/0 > /dev/null
done
```

### 7.3 View Graph

In Kiali UI:

-   Namespace: `bookinfo`
-   Graph Type: `Service Graph` or `Workload Graph`
-   Enable:
    -   Security
    -   Traffic Animation
    -   mTLS
![Kiali Traffic Graph](mTLS-images/2026-01-27_16-53-45.PNG)

You should see:

-   productpage → details with **lock icon** (mTLS)
![productpage->details](mTLS-images/2026-01-27_16-55-07.PNG)
-   reviews → details with **error rate / red edge**
![productpage->details](mTLS-images/2026-01-27_16-56-18.PNG)


------------------------------------------------------------------------

## PoC Summary Commands (Copy / Paste)

```powershell 
# Show STRICT mTLS
oc get peerauthentication -n bookinfo

# Show SPIFFE ID for productpage
PP=$(oc get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
oc exec -n bookinfo $PP -c istio-proxy --   curl -s http://localhost:15000/certs | grep spiffe

# Show AuthorizationPolicy
oc get authorizationpolicy -n bookinfo details-allow-only-productpage -o yaml

# DENIED example
REVIEWS=$(oc get pod -n bookinfo -l app=reviews -o jsonpath='{.items[0].metadata.name}')
oc exec -n bookinfo $REVIEWS -c reviews --   curl -sS -o /dev/null -w "HTTP=%{http_code}
" http://details:9080/details/0

# ALLOWED example
oc exec -n bookinfo spiffe-test-productpage -c sleep --   curl -sS -o /dev/null -w "HTTP=%{http_code}
" http://details:9080/details/0
```

------------------------------------------------------------------------

## Cleanup

```powershell 
oc delete pod -n bookinfo spiffe-test-productpage
oc delete authorizationpolicy -n bookinfo details-allow-only-productpage
oc delete peerauthentication -n bookinfo default
```

------------------------------------------------------------------------

## Architecture (Conceptual)

    productpage ──mTLS(SPIFFE)──► details
    reviews     ──mTLS(SPIFFE)──► details   ❌ denied by policy

------------------------------------------------------------------------

End of Guide.
