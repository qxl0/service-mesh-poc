# OSSM 3 (Service Mesh 3) PoC — Traffic Shifting (Canary / Weighted Routing) with Bookinfo

This PoC guide shows how to implement and verify **traffic shifting** (canary / weighted routing) in **Red Hat OpenShift Service Mesh 3** on **OpenShift Local (CRC)** using the **Bookinfo** sample.

> Assumptions: You already have OSSM 3 running, Bookinfo deployed, and you can access Bookinfo via an OpenShift Route.  
> Namespaces used below: `istio-system`, `istio-gateway`, `bookinfo`.

---

## Roles

- **developer**: all resources in `bookinfo` namespace (DestinationRule / VirtualService) + traffic generation
- **kubeadmin**: only needed for cluster/operator install, control plane setup, and installing addons

This guide focuses on **developer** actions only.

---

## What you will build

You will route traffic from `productpage` → `reviews` using these patterns:

1. **Weighted routing** (e.g., 90/10 → 50/50 → 100/0)
2. Optional: **Header-based routing** (A/B style) for controlled experiments

Verification will be done using **Kiali metrics/graph** (recommended).

---

## 0) Quick sanity checks (developer)

### 0.1 Confirm reviews versions exist and have sidecars
```powershell
oc get pods -n bookinfo -l app=reviews
```
Expected: `reviews-v1`, `reviews-v2`, `reviews-v3` are Running and **2/2**.

### 0.2 Confirm namespace injection label exists
```powershell
oc get ns bookinfo --show-labels
```
Expected label: `istio-injection=enabled` (or your mesh’s injection method).

---

## 1) Create DestinationRule for `reviews` subsets (developer)

Istio uses `DestinationRule` subsets to map versions using labels like `version=v1`.

Apply:

```powershell
@"
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: bookinfo
spec:
  host: reviews.bookinfo.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
"@ | oc apply -f -
```

Verify:

```powershell
oc -n bookinfo get destinationrule reviews
oc -n bookinfo get destinationrule reviews -o yaml | Select-String -Pattern "host:|subsets:|name: v|version:"
```

Also verify pods carry matching labels:

```powershell
oc -n bookinfo get pods -l app=reviews --show-labels
```

---

## 2) Weighted Routing (Canary) with VirtualService (developer)

### 2.1 Start with 90% v1 / 10% v2

```powershell
@"
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
"@ | oc apply -f -
```

Verify:

```powershell
oc -n bookinfo get virtualservice reviews
oc -n bookinfo get virtualservice reviews -o yaml | Select-String -Pattern "subset:|weight:"
```

### 2.2 Change to 50/50

```powershell
@"
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v2
      weight: 50
"@ | oc apply -f -
```

### 2.3 Promote canary to 100% v2 (full rollout)

```powershell
@"
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
      weight: 100
"@ | oc apply -f -
```

---

## 3) Generate traffic (developer)

> Traffic must exist for Kiali/Grafana graphs to show anything.

Use your Bookinfo route (example below matches CRC):
```powershell
$gw = "http://bookinfo-istio-gateway.apps-crc.testing"
1..300 | % { curl -s "$gw/productpage" | Out-Null }
```

Tip: run again after changing weights.

---

## 4) Verify traffic shifting in Kiali (recommended)

### 4.1 Traffic Graph (visual)
Kiali → **Traffic Graph**

- Namespace: `bookinfo`
- Graph Type: **Workload**
- Time range: **Last 5m**
- Enable overlay: **Traffic distribution** (if available)

Expected:
- At 90/10: edge to `reviews-v1` is much thicker / higher RPS than `reviews-v2`
- At 50/50: edges are similar
- At 100% v2: only `reviews-v2` shows traffic

### 4.2 Metrics (most reliable)
Kiali → **Workloads** → `reviews-v1` and `reviews-v2`

For each workload:
- Open **Inbound Metrics**
- Time range: **Last 5m**
- Reporter: **destination**
- Look at **Request rate**

Expected:
- 50/50 should show similar request rates between v1 and v2 over the same time window.

---

## 5) Optional experiment: Header-based routing (A/B)

Route `end-user: jason` to v2, everyone else to v1:

```powershell
@"
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
      weight: 100
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
"@ | oc apply -f -
```

Send test traffic:

```powershell
$gw = "http://bookinfo-istio-gateway.apps-crc.testing"
1..100 | % { curl -s -H "end-user: jason" "$gw/productpage" | Out-Null }
1..100 | % { curl -s "$gw/productpage" | Out-Null }
```

Verify in Kiali using **Workloads → Inbound Metrics** for `reviews-v2` vs `reviews-v1`.

> Note: Envoy access logs are not always enabled to stdout by default, so metrics are the preferred proof.

---

## 6) Cleanup / Reset

To remove the `reviews` routing rules:

```powershell
oc -n bookinfo delete virtualservice reviews
oc -n bookinfo delete destinationrule reviews
```

---

## Troubleshooting

### A) “No traffic shown in Kiali”
- Generate more traffic (300–1000 requests)
- Set time range to **Last 5m**
- Ensure Bookinfo route is correct and reachable

### B) “Only v1 gets traffic”
- Check VirtualService spec:
  ```powershell
  oc -n bookinfo get virtualservice reviews -o yaml
  ```
- Check subsets match pod labels:
  ```powershell
  oc -n bookinfo get destinationrule reviews -o yaml
  oc -n bookinfo get pods -l app=reviews --show-labels
  ```

### C) Sidecars not injected (pods are 1/1 instead of 2/2)
- Ensure namespace label `istio-injection=enabled`
- Restart deployments:
  ```powershell
  oc -n bookinfo rollout restart deploy/reviews-v1
  oc -n bookinfo rollout restart deploy/reviews-v2
  ```

---

## Next experiments
- Fault injection (delay / abort)
- Retries / timeouts
- mTLS STRICT + lock icons
- AuthorizationPolicy (RBAC)
