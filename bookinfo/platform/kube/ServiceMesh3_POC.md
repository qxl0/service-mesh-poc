# OpenShift Local (CRC) — Service Mesh 3 PoC + Bookinfo + Gateway + Kiali

This guide is based on what we actually did in our session (Service Mesh **3.x / Sail operator**, Bookinfo sample, custom ingress gateway via **gateway injection**, OpenShift Route, then **Kiali** with in-mesh Prometheus).

> **Conventions**
> - **Role: kubeadmin** = cluster-admin tasks (cluster-scoped objects, operators, namespace labels, some routes, etc.)
> - **Role: developer** = app namespace tasks (deploying Bookinfo, Istio config objects in app namespace, generating traffic)

---

## 0) CRC sizing (optional but recommended)

If you haven’t sized CRC yet and plan to run Service Mesh + Kiali comfortably:

- memory: **24GB**
- cpus: **8**
- disk: **120GB**

PowerShell:
```powershell
crc config set memory 24576
crc config set cpus 8
crc config set disk-size 120
crc stop
crc start
```

1) Install Service Mesh 3 Operator

Role: kubeadmin
Use Web Console:

Operators → OperatorHub

Search: Red Hat OpenShift Service Mesh 3 Operator

Install (typically into openshift-operators)

2) Create control plane projects

You already had these created, but listing for completeness.

Role: developer or kubeadmin

oc new-project istio-system
oc new-project istio-cni

3) Create Service Mesh 3 control plane (Sail CRs)

Role: kubeadmin (recommended)

3.1 Create Istio in istio-system