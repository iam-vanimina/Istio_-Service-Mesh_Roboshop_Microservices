# 🚀 Istio Ambient Mesh – Roboshop

## 📌 Overview
```
Istio offical: https://istio.io/latest/docs/overview/quickstart/
```
This project demonstrates **Istio Ambient Mode** with the Kubernetes-based **Roboshop microservices application**.

Istio Ambient Mesh provides service-mesh capabilities without requiring an Envoy sidecar inside every application pod.

Instead, Ambient Mesh uses:

* **ztunnel** — lightweight Layer 4 proxy
* **Waypoint Proxy** — Layer 7 proxy when required
* **Istiod** — Istio control plane
* **Gateway** — ingress traffic management
* **Kiali** — service-mesh visualization
* **Prometheus** — metrics
* **Grafana** — dashboards
* **Loki + Alloy** — logs
* **OpenTelemetry + Tempo** — distributed tracing

---

# 🏗️ Ambient Mesh Architecture

```text
                         ┌─────────────────────┐
                         │       Browser       │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Istio Gateway     │
                         └──────────┬──────────┘
                                    │
                                    ▼
                    ┌──────────────────────────────┐
                    │       Roboshop Namespace     │
                    │                              │
                    │  ┌──────────┐  ┌──────────┐ │
                    │  │ Frontend │  │   Cart   │ │
                    │  └────┬─────┘  └────┬─────┘ │
                    │       │             │       │
                    │  ┌────▼─────────────▼─────┐ │
                    │  │        ztunnel         │ │
                    │  │       L4 Proxy         │ │
                    │  └────────────┬───────────┘ │
                    │               │             │
                    │        ┌──────▼──────┐      │
                    │        │   Waypoint  │      │
                    │        │     L7       │      │
                    │        └──────┬──────┘      │
                    │               │             │
                    │  ┌────────────▼──────────┐  │
                    │  │ Catalogue / Payment   │  │
                    │  │ Shipping / Dispatch   │  │
                    │  │ User / Other Apps     │  │
                    │  └───────────────────────┘  │
                    └──────────────────────────────┘

                              ▲
                              │
                    ┌─────────┴─────────┐
                    │      Istiod       │
                    │   Control Plane   │
                    └───────────────────┘
```

---

# 🔥 Sidecar vs Ambient

## Traditional Istio

```text
Pod
┌─────────────────────────────┐
│ Application                 │
│                             │
│ Envoy Sidecar               │
└─────────────────────────────┘
```

Every application pod receives an Envoy sidecar.

## Ambient Mesh

```text
Application Pod
┌─────────────────────────────┐
│ Application                 │
└─────────────────────────────┘
             │
             │
             ▼
        ┌─────────┐
        │ ztunnel │
        └─────────┘
```

The application pod does not need an Envoy sidecar.

For Layer 7 functionality:

```text
Application
     │
     ▼
  ztunnel
     │
     ▼
Waypoint Proxy
     │
     ▼
Destination
```

This allows Ambient Mesh to separate Layer 4 security from optional Layer 7 traffic management.

---

# 🧩 Main Components

| Component     | Responsibility             |
| ------------- | -------------------------- |
| Istiod        | Control plane              |
| ztunnel       | L4 secure proxy            |
| Waypoint      | L7 traffic processing      |
| Istio Gateway | Ingress/egress             |
| Kiali         | Service mesh visualization |
| Prometheus    | Metrics                    |
| Grafana       | Visualization              |
| Alloy         | Telemetry collection       |
| Loki          | Logs                       |
| OpenTelemetry | Tracing                    |
| Tempo         | Trace storage              |

---

# 📦 Install Istio

Download Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Add `istioctl` to PATH:

```bash
export PATH=$PWD/istio-*/bin:$PATH
```

Verify:

```bash
istioctl version
```

---

# 🚀 Install Istio Ambient Profile

Install Istio using the Ambient profile:

```bash
istioctl install --set profile=ambient -y
```

Verify:

```bash
kubectl get pods -n istio-system
```

You should see components such as:

```text
istiod
ztunnel
istio-ingressgateway
istio-egressgateway
```

Check:

```bash
kubectl get pods -n istio-system -o wide
```

---

# 🔍 Verify Ambient Installation

Run:

```bash
istioctl verify-install
```

Check the mesh:

```bash
istioctl ztunnel-config workloads
```

Check ztunnel configuration:

```bash
istioctl ztunnel-config all
```

Check services:

```bash
kubectl get svc -n istio-system
```

---

# 🏷️ Enable Ambient Mode

Label the Roboshop namespace:

```bash
kubectl label namespace roboshop \
  istio.io/dataplane-mode=ambient
```

Verify:

```bash
kubectl get namespace roboshop --show-labels
```

You should see:

```text
istio.io/dataplane-mode=ambient
```

Restarting application pods is generally not required simply to enroll the namespace into Ambient mode.

---

# 🔎 Verify Roboshop Workloads

```bash
kubectl get pods -n roboshop
```

Unlike sidecar mode, you should **not** expect:

```text
2/2
```

for every application pod.

For example:

```text
frontend-xxxxx      1/1
catalogue-xxxxx     1/1
cart-xxxxx          1/1
payment-xxxxx       1/1
shipping-xxxxx      1/1
dispatch-xxxxx      1/1
```

The application pods remain unchanged.

The traffic interception is provided by the Ambient data plane.

---

# 🔐 ztunnel

`ztunnel` provides the Layer 4 portion of Ambient Mesh.

Its responsibilities include:

* Secure workload-to-workload communication
* mTLS
* Identity
* TCP traffic handling
* Traffic interception
* L4 telemetry

Check ztunnel:

```bash
kubectl get pods -n istio-system -l app=ztunnel
```

Check logs:

```bash
kubectl logs -n istio-system \
  -l app=ztunnel
```

Check workloads:

```bash
istioctl ztunnel-config workloads
```

Example:

```text
NAMESPACE   POD                  IP          NODE
roboshop    frontend-xxxxx      10.1.1.10   desktop-worker
roboshop    catalogue-xxxxx     10.1.1.11   desktop-worker
roboshop    cart-xxxxx          10.1.1.12   desktop-worker
roboshop    payment-xxxxx       10.1.1.13   desktop-worker
```

---

# 🌐 Waypoint Proxy

Ambient mode uses **waypoint proxies** when Layer 7 processing is required.

Examples:

* HTTP routing
* Authorization policies
* HTTP retries
* HTTP timeouts
* Traffic splitting
* Header manipulation
* L7 observability

Create a waypoint:

```bash
istioctl waypoint apply \
  -n roboshop \
  --name roboshop-waypoint
```

Verify:

```bash
kubectl get gateway -n roboshop
```

Check waypoint pods:

```bash
kubectl get pods -n roboshop
```

---

# 🛣️ HTTP Traffic Through Waypoint

A simplified flow:

```text
Frontend
   │
   ▼
ztunnel
   │
   ▼
Waypoint
   │
   ▼
Catalogue
   │
   ▼
ztunnel
```

The waypoint handles Layer 7 policies while ztunnel handles the secure L4 transport.

---

# 🔐 Ambient mTLS

Ambient Mesh provides encrypted service-to-service communication using mTLS.

Traffic:

```text
Frontend
   │
   │ mTLS
   ▼
ztunnel
   │
   │ mTLS
   ▼
ztunnel
   │
   ▼
Catalogue
```

Check configuration:

```bash
istioctl ztunnel-config workloads
```

You can also inspect the ztunnel logs:

```bash
kubectl logs \
  -n istio-system \
  -l app=ztunnel
```

create service account frontend in namespace roboshop
```

venka@Think-VVRAM MINGW64 ~
$ kubectl create sa frontend -n roboshop

serviceaccount/frontend created

```
---

# 🔒 AuthorizationPolicy

Ambient mode supports Istio authorization policies.

Example:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalogue-policy
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: catalogue

  action: ALLOW

  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/roboshop/sa/frontend
```

Apply:

```bash
kubectl apply -f authorization-policy.yaml
```

For Layer 7 authorization, ensure the traffic is routed through an appropriate waypoint.

---

# 🚦 Traffic Management

Ambient Mesh can provide advanced traffic management through waypoint proxies.

Example:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalogue
  namespace: roboshop
spec:
  hosts:
    - catalogue
  http:
    - route:
        - destination:
            host: catalogue
            subset: v1 
          weight: 90

        - destination:
            host: catalogue
            subset: v2
          weight: 10
```

This allows controlled traffic splitting.

---

# 🌐 Istio Gateway

Create an Istio Gateway:

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: roboshop-gateway
  namespace: roboshop
spec:
  selector:
    istio: ingressgateway

  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP

      hosts:
        - "*"
```

Apply:

```bash
kubectl apply -f gateway.yaml
```

---

# 🔀 VirtualService

Route traffic to the frontend:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: frontend
  namespace: roboshop
spec:
  hosts:
    - "*"

  gateways:
    - roboshop-gateway

  http:
    - route:
        - destination:
            host: frontend
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f virtual-service.yaml
```

---

# 📊 Kiali

Kiali provides a visual representation of the service mesh.

Check Kiali:

```bash
kubectl get pods -n istio-system
```

Port-forward:

```bash
kubectl port-forward \
  svc/kiali \
  -n istio-system \
  20001:20001
```

Open:

```text
http://localhost:20001
```

Kiali can show:

```text
Frontend
   │
   ├── Cart
   ├── Catalogue
   └── User
          │
          ▼
       Database

Cart
 │
 ├── Payment
 │
 └── Shipping
        │
        ▼
     Dispatch
```

---

# 📈 Prometheus

Istio telemetry can be collected by Prometheus.

Check:

```bash
kubectl get pods -n monitoring
```

Example metric:

```promql
istio_requests_total
```

Request rate:

```promql
sum(
  rate(
    istio_requests_total[5m]
  )
)
```

Roboshop traffic:

```promql
sum(
  rate(
    istio_requests_total{
      destination_service_namespace="roboshop"
    }[5m]
  )
)
```

HTTP 5xx:

```promql
sum(
  rate(
    istio_requests_total{
      destination_service_namespace="roboshop",
      response_code=~"5.."
    }[5m]
  )
)
```

---

# 📊 Grafana

Recommended dashboards:

```text
Istio Mesh
Istio Service
Istio Workload
Istio Ambient
Roboshop Application
Kubernetes Cluster
```

Important metrics:

```text
Request Rate
Request Duration
HTTP 4xx
HTTP 5xx
TCP Connections
Workload Traffic
Service Dependencies
mTLS Traffic
```

---

# 📝 Alloy + Loki

Grafana Alloy collects application logs and sends them to Loki.

Example LogQL:

```logql
{namespace="roboshop"}
```

Payment:

```logql
{namespace="roboshop", app="payment"}
```

Catalogue:

```logql
{namespace="roboshop", app="catalogue"}
```

Cart:

```logql
{namespace="roboshop", app="cart"}
```

Frontend:

```logql
{namespace="roboshop", app="frontend"}
```

Errors:

```logql
{namespace="roboshop"} |= "ERROR"
```

---

# 🔭 OpenTelemetry + Tempo

The target tracing architecture:

```text
Roboshop Application
        │
        ▼
      Envoy
        │
        ▼
 OpenTelemetry
        │
        ▼
      Tempo
        │
        ▼
     Grafana
```

Combined with logs:

```text
                 Grafana
                /       \
               /         \
            Loki         Tempo
             ▲             ▲
             │             │
           Alloy      OpenTelemetry
             ▲             ▲
             │             │
       Application       Istio
```

This enables correlation between:

```text
Trace
  ↓
Span
  ↓
Service
  ↓
Pod
  ↓
Application Log
```

---

# 🧭 Complete Observability Architecture

```text
                         ┌───────────────┐
                         │    Grafana    │
                         └───────┬───────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
         Prometheus            Loki               Tempo
              ▲                  ▲                  ▲
              │                  │                  │
              │                Alloy          OpenTelemetry
              │                  ▲                  ▲
              │                  │                  │
              └──────────────┬───┴──────────────────┘
                             │
                        Istio Ambient
                             │
                   ┌─────────┴─────────┐
                   │                   │
                ztunnel             Waypoint
                   │                   │
                   └─────────┬─────────┘
                             │
                       Roboshop Apps
                             │
       ┌──────────┬──────────┼──────────┬──────────┐
       ▼          ▼          ▼          ▼          ▼
   Frontend     Cart     Catalogue   Payment   Shipping
                                                    │
                                                    ▼
                                                Dispatch
```

---

# 🔍 Ambient Troubleshooting

## Check Istio

```bash
istioctl version
```

## Check installation

```bash
istioctl verify-install
```

## Analyze configuration

```bash
istioctl analyze -A
```

## Check ztunnel

```bash
kubectl get pods -n istio-system -l app=ztunnel
```

## Check ztunnel configuration

```bash
istioctl ztunnel-config all
```

## Check workloads

```bash
istioctl ztunnel-config workloads
```

## Check services

```bash
istioctl ztunnel-config services
```

## Check certificates

```bash
istioctl ztunnel-config certificates
```

## Check waypoint

```bash
kubectl get pods -A | grep waypoint
```

## Check Istiod logs

```bash
kubectl logs \
  deployment/istiod \
  -n istio-system
```

## Check ztunnel logs

```bash
kubectl logs \
  -n istio-system \
  -l app=ztunnel
```

---

# 🧪 Validation Checklist

```text
☑ Kubernetes running
☑ Istio installed
☑ Ambient profile enabled
☑ Istiod running
☑ ztunnel running
☑ Roboshop namespace enrolled
☑ Application pods have no Envoy sidecars
☑ Workloads visible in ztunnel
☑ mTLS enabled
☑ Waypoint configured where required
☑ Gateway configured
☑ VirtualService configured
☑ AuthorizationPolicy configured
☑ Kiali configured
☑ Prometheus configured
☑ Grafana configured
☑ Alloy configured
☑ Loki configured
☑ OpenTelemetry configured
☑ Tempo configured
```

---

# ⚡ Important Difference

### Sidecar Mode

```text
                 Istiod
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
     Envoy      Envoy       Envoy
       │          │           │
      App        App         App
```

### Ambient Mode

```text
                 Istiod
                   │
                   ▼
                ztunnel
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
      App         App         App
       │           │           │
       └───────────┼───────────┘
                   │
              Waypoint
             (when L7 needed)
```

The major architectural difference is that Ambient Mesh moves common mesh functionality out of individual application pods and into the node-level ztunnel layer, with waypoint proxies providing optional Layer 7 processing.

---

# 🎯 Key Learning Outcomes

This project demonstrates:

* Kubernetes service mesh
* Istio Ambient Mode
* ztunnel
* Waypoint Proxy
* mTLS
* Service identity
* AuthorizationPolicy
* Gateway
* VirtualService
* DestinationRule
* Traffic management
* Kiali
* Prometheus
* Grafana
* Grafana Alloy
* Loki
* OpenTelemetry
* Grafana Tempo
* Distributed tracing
* Logs and trace correlation
* Kubernetes observability

---

# 🚀 Future Enhancements

* [ ] Ambient canary deployment
* [ ] Waypoint-based L7 routing
* [ ] Fault injection
* [ ] Circuit breaking
* [ ] Retry policies
* [ ] Timeout policies
* [ ] Rate limiting
* [ ] Egress policies
* [ ] ServiceEntry
* [ ] OpenTelemetry Collector
* [ ] Tempo trace-to-log correlation
* [ ] Istio SLO dashboards
* [ ] Alertmanager
* [ ] Argo CD GitOps for Istio
* [ ] Multi-cluster Ambient Mesh

---

# 👨‍💻 Author

**Venkata Ram Vanimina**

DevOps | AWS | Azure | Kubernetes | Terraform | Istio | GitOps | Observability

---

# ⭐ Project Summary

This project implements a Kubernetes-based Roboshop platform using **Istio Ambient Mesh**.

Instead of injecting an Envoy sidecar into every application pod, Ambient Mesh uses:

```text
ztunnel
   +
Waypoint Proxy
   +
Istiod
```

to provide service-mesh functionality.

The complete platform provides:

```text
              USER
               │
               ▼
        Istio Gateway
               │
               ▼
          ztunnel
               │
               ▼
         Waypoint L7
               │
               ▼
       Roboshop Services
               │
       ┌───────┼────────┐
       ▼       ▼        ▼
    Metrics   Logs    Traces
       │       │        │
       ▼       ▼        ▼
 Prometheus   Loki    Tempo
       │       │        │
       └───────┼────────┘
               ▼
            Grafana
```

This provides a complete **Kubernetes Ambient Service Mesh + Security + Traffic Management + Observability** implementation.
