# 🚀 Istio Service Mesh – Roboshop Microservices

## 📌 Overview

This project demonstrates how to integrate **Istio Service Mesh** with a Kubernetes-based **Roboshop microservices application**.

Istio provides a dedicated service-mesh layer for:

* 🔐 Service-to-service security
* 🔄 Traffic management
* ⚖️ Load balancing
* 🛡️ Mutual TLS (mTLS)
* 📊 Metrics and observability
* 🔍 Distributed tracing
* 📝 Access logging
* 🚦 Canary and weighted traffic routing
* 🔒 Authorization policies
* 🌐 Ingress and egress traffic control

Istio works transparently with applications by using Envoy proxies to intercept and manage service traffic.

---

# 🏗️ Architecture

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
                         ┌─────────────────────┐
                         │      Frontend       │
                         │   + Envoy Sidecar   │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
       ┌────────────┐        ┌────────────┐        ┌────────────┐
       │   Cart     │        │ Catalogue  │        │   User     │
       │ + Envoy    │        │ + Envoy    │        │ + Envoy    │
       └─────┬──────┘        └─────┬──────┘        └────────────┘
             │                     │
             ▼                     ▼
       ┌────────────┐        ┌────────────┐
       │  Shipping  │        │  Payment   │
       │ + Envoy    │        │ + Envoy    │
       └─────┬──────┘        └────────────┘
             │
             ▼
       ┌────────────┐
       │  Dispatch  │
       │ + Envoy    │
       └────────────┘

                ┌─────────────────────────┐
                │       Istiod            │
                │ Control Plane           │
                │                         │
                │ • Service Discovery     │
                │ • Configuration         │
                │ • Certificates          │
                │ • Traffic Policies      │
                └─────────────────────────┘

        Observability
               │
       ┌───────┼────────┬───────────┐
       ▼       ▼        ▼           ▼
   Prometheus Grafana  Loki       Tempo
                         ▲           ▲
                         │           │
                       Alloy     OpenTelemetry
```

Istio separates the mesh into a **control plane** and **data plane**. Envoy proxies handle application traffic while `istiod` manages configuration, service discovery, and certificates.

---

# 🛠️ Technology Stack

| Technology     | Purpose                       |
| -------------- | ----------------------------- |
| Kubernetes     | Container orchestration       |
| Docker Desktop | Local Kubernetes environment  |
| Istio          | Service Mesh                  |
| Envoy          | Data-plane proxy              |
| Istiod         | Istio control plane           |
| Kiali          | Service Mesh visualization    |
| Prometheus     | Metrics                       |
| Grafana        | Dashboards                    |
| Grafana Alloy  | Telemetry collection          |
| Loki           | Logs                          |
| OpenTelemetry  | Distributed tracing           |
| Grafana Tempo  | Trace storage                 |
| Helm           | Kubernetes package management |
| Argo CD        | GitOps deployment             |
| GitHub         | Source control                |

---

# 📂 Project Structure

```text
roboshop/
│
├── CD/
│   ├── frontend-argocd/
│   ├── catalogue-argocd/
│   ├── cart-argocd/
│   ├── user-argocd/
│   ├── payment-argocd/
│   ├── shipping-argocd/
│   └── dispatch-argocd/
│
├── istio/
│   ├── gateway.yaml
│   ├── virtual-service.yaml
│   ├── destination-rules.yaml
│   ├── peer-authentication.yaml
│   ├── authorization-policy.yaml
│   └── telemetry.yaml
│
└── monitoring/
    ├── prometheus/
    ├── grafana/
    ├── alloy/
    ├── loki/
    └── tempo/
```

---

# 1️⃣ Install Istio

Download and install Istio:

```bash
istioctl install --set profile=demo -y
```

Verify:

```bash
istioctl version
```

Check the Istio namespace:

```bash
kubectl get pods -n istio-system
```

Expected components include:

```text
istiod
istio-ingressgateway
istio-egressgateway
```

Istio can be installed with `istioctl`, and the official documentation provides multiple installation profiles and deployment models.

---

# 2️⃣ Enable Istio Injection

Enable automatic Envoy sidecar injection for the Roboshop namespace:

```bash
kubectl label namespace roboshop istio-injection=enabled
```

Verify:

```bash
kubectl get namespace roboshop --show-labels
```

Restart the applications:

```bash
kubectl rollout restart deployment -n roboshop
```

Verify sidecars:

```bash
kubectl get pods -n roboshop
```

You should see:

```text
2/2
```

For example:

```text
payment-xxxxx       2/2   Running
cart-xxxxx          2/2   Running
catalogue-xxxxx     2/2   Running
frontend-xxxxx      2/2   Running
```

The second container is the Envoy sidecar.

---

# 3️⃣ Verify Istio Proxies

Check the proxy status:

```bash
istioctl proxy-status
```

Example:

```text
NAME                                      CDS       LDS       EDS       RDS
frontend-xxxxx.roboshop                   SYNCED    SYNCED    SYNCED    SYNCED
cart-xxxxx.roboshop                       SYNCED    SYNCED    SYNCED    SYNCED
catalogue-xxxxx.roboshop                  SYNCED    SYNCED    SYNCED    SYNCED
payment-xxxxx.roboshop                    SYNCED    SYNCED    SYNCED    SYNCED
```

Check proxy configuration:

```bash
istioctl proxy-config cluster <POD_NAME> -n roboshop
```

---

# 4️⃣ Istio Gateway

Create an ingress gateway:

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

# 5️⃣ Virtual Service

Route external traffic to the frontend:

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

# 6️⃣ Destination Rules

Destination rules define policies applied after traffic is routed to a service.

Example:

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: catalogue
  namespace: roboshop
spec:
  host: catalogue
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 5s
      baseEjectionTime: 30s
```

Apply:

```bash
kubectl apply -f destination-rules.yaml
```

---

# 7️⃣ Traffic Management

Istio can control traffic using:

* VirtualService
* DestinationRule
* Gateway
* ServiceEntry
* Sidecar

Example weighted routing:

```yaml
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

This can be used for controlled canary deployments and staged rollouts. Istio supports advanced routing, retries, failover, and traffic splitting.

---

# 8️⃣ Mutual TLS

Check the current authentication policies:

```bash
kubectl get peerauthentication -A
```

Example namespace-wide policy:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: roboshop
spec:
  mtls:
    mode: STRICT
```

Apply:

```bash
kubectl apply -f peer-authentication.yaml
```

Verify:

```bash
istioctl x describe pod <POD_NAME> -n roboshop
```

Istio uses workload identities and certificates to provide service-to-service authentication and mTLS encryption.

---

# 9️⃣ Authorization Policy

Example policy allowing only the frontend to call the catalogue service:

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
              - "cluster.local/ns/roboshop/sa/frontend"
```

Apply:

```bash
kubectl apply -f authorization-policy.yaml
```

Istio authorization policies can enforce workload-level access control using `ALLOW`, `DENY`, and `CUSTOM` actions.

---

# 🔟 Kiali

Kiali provides a graphical view of the service mesh.

Check Kiali:

```bash
kubectl get pods -n istio-system
```

Port-forward:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Open:

```text
http://localhost:20001
```

Kiali can be used to visualize:

```text
Frontend
   ↓
Cart
   ↓
Catalogue
   ↓
MongoDB

Frontend
   ↓
User

Cart
   ↓
Payment

Cart
   ↓
Shipping
   ↓
Dispatch
```

---

# 1️⃣1️⃣ Prometheus

Istio exposes service-level telemetry that can be collected by Prometheus. Istio metrics cover traffic, latency, errors, and saturation.

Check Prometheus:

```bash
kubectl get pods -n monitoring
```

Example PromQL:

```promql
istio_requests_total
```

HTTP requests:

```promql
sum(rate(istio_requests_total[5m]))
```

Errors:

```promql
sum(rate(istio_requests_total{
  response_code=~"5xx"
}[5m]))
```

Success rate:

```promql
sum(rate(istio_requests_total{
  response_code=~"2xx"
}[5m]))
/
sum(rate(istio_requests_total[5m]))
```

---

# 1️⃣2️⃣ Grafana

Istio telemetry can be visualized using Grafana dashboards.

Recommended dashboards:

```text
Istio Mesh Dashboard
Istio Service Dashboard
Istio Workload Dashboard
Istio Performance Dashboard
```

Example metrics:

```text
Request Rate
Request Duration
HTTP Errors
TCP Connections
Service Dependencies
Workload Health
```

---

# 1️⃣3️⃣ Loki + Grafana Alloy

Grafana Alloy collects Kubernetes application logs and sends them to Loki.

Example Loki query:

```logql
{namespace="roboshop"}
```

Payment logs:

```logql
{namespace="roboshop", app="payment"}
```

Cart logs:

```logql
{namespace="roboshop", app="cart"}
```

Frontend logs:

```logql
{namespace="roboshop", app="frontend"}
```

Error logs:

```logql
{namespace="roboshop"} |= "ERROR"
```

Warning logs:

```logql
{namespace="roboshop"} |= "WARN"
```

Istio can also generate access logs containing source, destination, request and response information.

---

# 1️⃣4️⃣ OpenTelemetry + Grafana Tempo

For distributed tracing, this project can integrate:

```text
Roboshop
   │
   ▼
Istio / Envoy
   │
   ▼
OpenTelemetry
   │
   ▼
Grafana Tempo
   │
   ▼
Grafana
```

Tracing helps identify:

* Request flow
* Service dependencies
* Latency
* Failed requests
* Slow services
* Cross-service communication

Istio supports distributed tracing through Envoy proxies, including OpenTelemetry-compatible tracing backends.

---

# 1️⃣5️⃣ Istio + Loki + Tempo

The target observability architecture is:

```text
                    ┌───────────────┐
                    │   Grafana     │
                    └───────┬───────┘
                            │
                ┌───────────┴───────────┐
                │                       │
                ▼                       ▼
             Loki                    Tempo
                ▲                       ▲
                │                       │
             Alloy                OpenTelemetry
                ▲                       ▲
                │                       │
          Kubernetes Pods        Istio / Envoy
                │                       │
                └───────────┬───────────┘
                            │
                     Roboshop Services
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
Application Logs
```

---

# 1️⃣6️⃣ Useful Istio Commands

### Check Istio installation

```bash
istioctl version
```

### Validate configuration

```bash
istioctl analyze -A
```

### Check proxies

```bash
istioctl proxy-status
```

### Check proxy configuration

```bash
istioctl proxy-config cluster <POD> -n roboshop
```

```bash
istioctl proxy-config routes <POD> -n roboshop
```

```bash
istioctl proxy-config listeners <POD> -n roboshop
```

### Check Istio resources

```bash
kubectl get gateway -A
kubectl get virtualservice -A
kubectl get destinationrule -A
kubectl get peerauthentication -A
kubectl get authorizationpolicy -A
```

### Analyze configuration

```bash
istioctl analyze -n roboshop
```

---

# 1️⃣7️⃣ Troubleshooting

### Check Envoy sidecar

```bash
kubectl get pod <POD_NAME> -n roboshop
```

Expected:

```text
2/2 Running
```

### Check Envoy logs

```bash
kubectl logs <POD_NAME> -n roboshop -c istio-proxy
```

### Check Istiod

```bash
kubectl logs deployment/istiod -n istio-system
```

### Check Kiali

```bash
kubectl logs deployment/kiali -n istio-system
```

### Check configuration problems

```bash
istioctl analyze -A
```

### Check service endpoints

```bash
kubectl get endpoints -n roboshop
```

### Check services

```bash
kubectl get svc -n roboshop
```

---

# 1️⃣8️⃣ Verification Checklist

```text
☑ Kubernetes cluster running
☑ Istio installed
☑ Istiod running
☑ Istio ingress gateway running
☑ Roboshop namespace created
☑ Istio injection enabled
☑ Envoy sidecars injected
☑ Gateway configured
☑ VirtualService configured
☑ DestinationRules configured
☑ mTLS configured
☑ AuthorizationPolicy configured
☑ Kiali configured
☑ Prometheus configured
☑ Grafana configured
☑ Alloy configured
☑ Loki configured
☑ OpenTelemetry configured
☑ Tempo configured
☑ Trace-to-log correlation configured
```

---

# 🎯 Key Learning Outcomes

This project demonstrates practical implementation of:

* Kubernetes Service Mesh
* Istio architecture
* Envoy sidecars
* Service discovery
* HTTP traffic management
* Gateway configuration
* VirtualService
* DestinationRule
* Canary deployments
* mTLS
* AuthorizationPolicy
* Kiali
* Prometheus
* Grafana
* Grafana Alloy
* Loki
* OpenTelemetry
* Grafana Tempo
* Distributed tracing
* Logs and trace correlation
* Microservices observability

---

# 📊 Observability Stack

```text
                    ROBOSHOP
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
      Metrics         Logs          Traces
        │              │              │
        ▼              ▼              ▼
   Prometheus         Loki          Tempo
        │              │              │
        └──────────────┼──────────────┘
                       ▼
                    Grafana
                       │
                       ▼
              Complete Observability
```

---

# 🔐 Security

The Istio implementation provides:

```text
Workload Identity
       │
       ▼
      mTLS
       │
       ▼
Authentication
       │
       ▼
AuthorizationPolicy
       │
       ▼
Secure Service-to-Service Communication
```

---

# 🚀 Future Enhancements

* [ ] Istio Gateway API
* [ ] Canary deployments
* [ ] Blue/Green deployments
* [ ] Fault injection
* [ ] Circuit breaking
* [ ] Retry policies
* [ ] Request timeouts
* [ ] Rate limiting
* [ ] Egress control
* [ ] External services using ServiceEntry
* [ ] OpenTelemetry Collector
* [ ] Tempo trace-to-log correlation
* [ ] Istio SLO dashboards
* [ ] Alertmanager integration
* [ ] Argo CD GitOps for Istio configuration

---

# 👨‍💻 Author

**Venkata Ram Vanimina**

DevOps / AWS / Azure / Kubernetes / Terraform / Istio

---

# 📚 References

* Istio Documentation: https://istio.io/
* Istio Architecture: https://istio.io/latest/docs/ops/deployment/architecture/
* Istio Observability: https://istio.io/latest/docs/concepts/observability/
* Istio Security: https://istio.io/latest/docs/concepts/security/
* Istio GitHub: https://github.com/istio/istio

---

## ⭐ Project Summary

This project demonstrates a production-oriented Kubernetes microservices environment using **Istio Service Mesh** for traffic management, security, and observability, integrated with **Kiali, Prometheus, Grafana, Alloy, Loki, OpenTelemetry, and Tempo**.

The resulting platform provides visibility from:

```text
User Request
     ↓
Istio Gateway
     ↓
Envoy
     ↓
Roboshop Microservices
     ↓
Metrics + Logs + Traces
     ↓
Prometheus + Loki + Tempo
     ↓
Grafana
```

This creates a complete **service mesh and observability platform for Kubernetes microservices**.
