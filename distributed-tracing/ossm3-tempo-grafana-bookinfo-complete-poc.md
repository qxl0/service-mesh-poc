
# OpenShift Service Mesh 3 + Tempo + Grafana + Bookinfo
## Distributed Tracing & Service Graph Complete PoC Guide (PowerShell)

---

## Architecture
Bookinfo -> Istio Proxy -> OTLP -> OpenTelemetry Collector -> Tempo -> Metrics Generator -> Prometheus -> Grafana

---

# PART 1 — Distributed Tracing

## Namespaces
```powershell
oc new-project bookinfo
oc new-project tracing-system
oc label namespace bookinfo istio-injection=enabled --overwrite
```

## Deploy Bookinfo
```powershell
oc -n bookinfo apply -f bookinfo.yaml
oc -n bookinfo apply -f bookinfo-gateway.yaml
```

## Deploy OpenTelemetry Collector
```powershell
oc apply -f otel-collector.yaml
```

## Enable Istio Telemetry
```powershell
oc apply -f telemetry.yaml
```

## Restart Workloads
```powershell
oc -n bookinfo rollout restart deploy/productpage-v1
```

## Generate Traffic
```powershell
$sp=(oc -n bookinfo get pod -l app=sleep -o jsonpath='{.items[0].metadata.name}')
1..50 | % { oc -n bookinfo exec $sp -c sleep -- sh -c "wget -qO- http://productpage:9080/productpage > /dev/null" }
```

## Add Tempo Datasource
URL: http://tempo-tempo.tracing-system.svc.cluster.local:3200

---

# PART 2 — Service Graph

## Enable Metrics Generator
```powershell
oc -n tracing-system patch tempomonolithic tempo --type merge -p '{ "spec": { "extraConfig": { "tempo": { "metrics_generator": { "processor": { "service_graphs": {}, "span_metrics": {} } } } } }'
```

Restart:
```powershell
oc -n tracing-system rollout restart statefulset tempo-tempo
```

## Configure Grafana
Tempo datasource -> Service Graph -> Prometheus

---

# Done
