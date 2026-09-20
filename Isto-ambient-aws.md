# RoboShop + Istio Ambient Service Mesh

Production-style Kubernetes service mesh implementation for the **RoboShop microservices application** using **Istio Ambient Mode**, including ztunnel, waypoint proxies, mTLS, L4/L7 AuthorizationPolicies, ServiceAccounts, and service-to-service security.

---

## Architecture

```text
                              Internet
                                  |
                                  v
                         +----------------+
                         | Istio Gateway  |
                         +-------+--------+
                                 |
                                 v
                            +---------+
                            |frontend |
                            +----+----+
                                 |
              +------------------+------------------+
              |                  |                  |
              v                  v                  v
         +---------+        +---------+        +---------+
         |catalogue|        |  cart   |        |  user   |
         +----+----+        +----+----+        +----+----+
              |                  |                  |
              v                  v                  v
          MongoDB              Redis              MySQL
                                                    ^
                                                    |
                                             +------+------+
                                             |             |
                                          payment       shipping
                                             |
                                             v
                                          RabbitMQ
                                             ^
                                             |
                                          dispatch


============================================================
                    ISTIO AMBIENT MESH
============================================================

                    +------------------+
                    |     Waypoint     |
                    |                  |
                    | L7 Authorization |
                    | HTTP Policies    |
                    +--------+---------+
                             |
                           HBONE
                             |
                         ztunnel
                             |
                       mTLS / L4
                             |
                         Workloads
```

---

# Why Istio Ambient?

Traditional Istio sidecar mode injects an Envoy proxy into every application pod.

Ambient mode separates the data plane into layers:

```text
Application
     |
     v
ztunnel
     |
     | L4 + mTLS + identity
     v
Waypoint
     |
     | L7 HTTP processing
     v
Destination
```

This means RoboShop workloads do not require an Envoy sidecar in every pod.

---

# Project Goals

This project demonstrates:

* Istio Ambient Mode
* ztunnel
* Waypoint proxy
* Kubernetes ServiceAccounts
* Workload identity
* STRICT mTLS
* L4 AuthorizationPolicy
* L7 AuthorizationPolicy
* Default deny
* Waypoint enforcement
* Service-to-service authorization
* Gateway API
* HTTP routing
* Observability
* Kubernetes security
* Terraform infrastructure
* AWS EKS
* CI/CD integration

---

# RoboShop Services

The example policy model covers:

```text
frontend
catalogue
cart
user
payment
shipping
dispatch
mongodb
redis
mysql
rabbitmq
```

---

# Repository Structure

```text
roboshop-istio-ambient/
│
├── terraform/
│   ├── versions.tf
│   ├── provider.tf
│   ├── variables.tf
│   ├── main.tf
│   ├── outputs.tf
│   ├── terraform.tfvars
│   │
│   └── modules/
│       ├── vpc/
│       ├── eks/
│       ├── iam/
│       └── istio/
│
├── kubernetes/
│   ├── namespace/
│   ├── serviceaccounts/
│   └── roboshop/
│
├── istio/
│   ├── ambient/
│   │   ├── peer-authentication.yaml
│   │   ├── waypoint.yaml
│   │   └── default-deny.yaml
│   │
│   ├── policies/
│   │   ├── 00-serviceaccounts.yaml
│   │   ├── 01-peer-authentication.yaml
│   │   ├── 02-default-deny.yaml
│   │   ├── 03-waypoint-enforcement.yaml
│   │   │
│   │   ├── frontend/
│   │   │   ├── catalogue.yaml
│   │   │   ├── cart.yaml
│   │   │   └── user.yaml
│   │   │
│   │   ├── catalogue/
│   │   │   └── mongodb.yaml
│   │   │
│   │   ├── cart/
│   │   │   ├── catalogue.yaml
│   │   │   └── redis.yaml
│   │   │
│   │   ├── user/
│   │   │   └── mysql.yaml
│   │   │
│   │   ├── payment/
│   │   │   ├── mysql.yaml
│   │   │   └── rabbitmq.yaml
│   │   │
│   │   ├── shipping/
│   │   │   ├── mysql.yaml
│   │   │   └── rabbitmq.yaml
│   │   │
│   │   └── dispatch/
│   │       └── rabbitmq.yaml
│   │
│   ├── gateway/
│   │   ├── gateway.yaml
│   │   └── httproutes.yaml
│   │
│   └── observability/
│       ├── telemetry.yaml
│       └── tracing.yaml
│
├── jenkins/
│   └── Jenkinsfile
│
└── README.md
```

---

# 1. AWS Infrastructure

The project uses a two-AZ AWS network:

```text
VPC
10.0.0.0/16

AZ-a
├── Public:  10.0.1.0/24
└── Private: 10.0.11.0/24

AZ-b
├── Public:  10.0.2.0/24
└── Private: 10.0.12.0/24
```

Architecture:

```text
                    Internet
                       |
                Internet Gateway
                  /          \
                 /            \
           Public-AZ-a    Public-AZ-b
                |              |
              NAT-A           NAT-B
                |              |
           Private-AZ-a   Private-AZ-b
                |              |
               EKS            EKS
```

Worker nodes run in private subnets.

---

# 2. Create the VPC

```bash
cd terraform

terraform init
```

Format:

```bash
terraform fmt -recursive
```

Validate:

```bash
terraform validate
```

Plan:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

---

# 3. Create the RoboShop Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: roboshop
  labels:
    istio.io/dataplane-mode: ambient
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

Verify:

```bash
kubectl get namespace roboshop --show-labels
```

Expected:

```text
istio.io/dataplane-mode=ambient
```

---

# 4. Install Istio Ambient

Add the Istio Helm repository:

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

Install the base chart:

```bash
helm install istio-base istio/base \
  -n istio-system \
  --create-namespace
```

Install Istiod:

```bash
helm install istiod istio/istiod \
  -n istio-system \
  --wait
```

Install Istio CNI:

```bash
helm install istio-cni istio/cni \
  -n istio-system \
  --set profile=ambient \
  --wait
```

Install ztunnel:

```bash
helm install ztunnel istio/ztunnel \
  -n istio-system \
  --wait
```

Verify:

```bash
kubectl get pods -n istio-system
```

---

# 5. Deploy the Waypoint

```bash
istioctl waypoint apply \
  -n roboshop \
  --enroll-namespace \
  --wait
```

Verify:

```bash
kubectl get gateway -n roboshop
```

Expected:

```text
NAME       CLASS            PROGRAMMED
waypoint   istio-waypoint   True
```

---

# 6. ServiceAccounts

Every RoboShop workload receives its own ServiceAccount.

```text
frontend
catalogue
cart
user
payment
shipping
dispatch
mongodb
redis
mysql
rabbitmq
```

Example:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend
  namespace: roboshop
```

Deployment:

```yaml
spec:
  template:
    spec:
      serviceAccountName: frontend
```

ServiceAccount identity becomes:

```text
cluster.local/ns/roboshop/sa/frontend
```

This identity is used by Istio AuthorizationPolicies.

---

# 7. STRICT mTLS

Create:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: roboshop-strict-mtls
  namespace: roboshop
spec:
  mtls:
    mode: STRICT
```

Apply:

```bash
kubectl apply -f peer-authentication.yaml
```

---

# 8. Default Deny

Create:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: default-deny
  namespace: roboshop
spec: {}
```

This establishes a deny-by-default authorization model.

Explicit ALLOW policies are then used for permitted communication.

---

# 9. Waypoint Enforcement

Waypoint enforcement prevents workloads from bypassing L7 authorization.

Example for catalogue:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalogue-require-waypoint
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
        - cluster.local/ns/roboshop/sa/waypoint
```

The traffic flow becomes:

```text
frontend
   |
   v
frontend ztunnel
   |
   | mTLS / HBONE
   v
waypoint
   |
   | L7 Authorization
   v
catalogue ztunnel
   |
   v
catalogue
```

---

# 10. Service-to-Service Authorization

## Frontend → Catalogue

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalogue-frontend
  namespace: roboshop
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: catalogue

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/frontend
```

---

## Frontend → Cart

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: cart-frontend
  namespace: roboshop
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: cart

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/frontend
```

---

## Frontend → User

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: user-frontend
  namespace: roboshop
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: user

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/frontend
```

---

## Cart → Catalogue

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalogue-cart
  namespace: roboshop
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: catalogue

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/cart
```

---

## Cart → Redis

Redis uses TCP, so use an L4 policy:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: redis-cart
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: redis

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/cart

    to:
    - operation:
        ports:
        - "6379"
```

---

## Catalogue → MongoDB

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: mongodb-catalogue
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: mongodb

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/catalogue

    to:
    - operation:
        ports:
        - "27017"
```

---

## User → MySQL

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: mysql-user
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: mysql

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/user

    to:
    - operation:
        ports:
        - "3306"
```

---

## Payment → MySQL

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: mysql-payment
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: mysql

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/payment

    to:
    - operation:
        ports:
        - "3306"
```

---

## Payment → RabbitMQ

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: rabbitmq-payment
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: rabbitmq

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/payment

    to:
    - operation:
        ports:
        - "5672"
```

---

## Shipping → MySQL

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: mysql-shipping
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: mysql

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/shipping

    to:
    - operation:
        ports:
        - "3306"
```

---

## Shipping → RabbitMQ

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: rabbitmq-shipping
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: rabbitmq

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/shipping

    to:
    - operation:
        ports:
        - "5672"
```

---

## Dispatch → RabbitMQ

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: rabbitmq-dispatch
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: rabbitmq

  action: ALLOW

  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/roboshop/sa/dispatch

    to:
    - operation:
        ports:
        - "5672"
```

---

# 11. Communication Matrix

| Source            | Destination | Protocol |  Port | Access |
| ----------------- | ----------- | -------- | ----: | ------ |
| Gateway           | frontend    | HTTP     |    80 | ALLOW  |
| frontend          | catalogue   | HTTP     |  8080 | ALLOW  |
| frontend          | cart        | HTTP     |  8080 | ALLOW  |
| frontend          | user        | HTTP     |  8080 | ALLOW  |
| cart              | catalogue   | HTTP     |  8080 | ALLOW  |
| cart              | redis       | TCP      |  6379 | ALLOW  |
| catalogue         | mongodb     | TCP      | 27017 | ALLOW  |
| user              | mysql       | TCP      |  3306 | ALLOW  |
| payment           | mysql       | TCP      |  3306 | ALLOW  |
| payment           | rabbitmq    | AMQP     |  5672 | ALLOW  |
| shipping          | mysql       | TCP      |  3306 | ALLOW  |
| shipping          | rabbitmq    | AMQP     |  5672 | ALLOW  |
| dispatch          | rabbitmq    | AMQP     |  5672 | ALLOW  |
| Any other traffic | Any         | Any      |   Any | DENY   |

> **Important:** Verify the actual RoboShop version's service ports and API paths before applying production policies. The matrix above represents the intended security model and should be aligned with the manifests actually deployed.

---

# 12. L4 vs L7 Policies

## L4 — ztunnel

Use L4 authorization for:

```text
ServiceAccount
Workload identity
Namespace
Port
TCP
```

Example:

```yaml
spec:
  selector:
    matchLabels:
      app: redis
```

---

## L7 — Waypoint

Use L7 authorization for:

```text
HTTP method
HTTP path
HTTP headers
HTTP traffic
```

Example:

```yaml
spec:
  targetRefs:
  - group: ""
    kind: Service
    name: catalogue

  rules:
  - to:
    - operation:
        methods:
        - GET
```

For ambient mode, L7 policies should target the appropriate Service or waypoint rather than being treated like traditional sidecar policies.

---

# 13. Testing

Check workloads:

```bash
kubectl get pods -n roboshop
```

Check ServiceAccounts:

```bash
kubectl get serviceaccounts -n roboshop
```

Check policies:

```bash
kubectl get authorizationpolicy -n roboshop
```

Check mTLS:

```bash
kubectl get peerauthentication -n roboshop
```

Check waypoint:

```bash
kubectl get gateway -n roboshop
```

Check ambient workloads:

```bash
istioctl ztunnel-config workloads
```

---

# 14. Allowed Traffic Test

Example:

```bash
kubectl exec deploy/frontend -n roboshop -- \
  curl -v http://catalogue:8080/
```

Expected:

```text
SUCCESS
```

Cart:

```bash
kubectl exec deploy/cart -n roboshop -- \
  curl -v http://catalogue:8080/
```

Expected:

```text
SUCCESS
```

---

# 15. Unauthorized Traffic Test

Payment attempting to access catalogue:

```bash
kubectl exec deploy/payment -n roboshop -- \
  curl -v http://catalogue:8080/
```

Expected:

```text
RBAC: access denied
```

Frontend attempting to access MySQL:

```bash
kubectl exec deploy/frontend -n roboshop -- \
  curl -v mysql:3306
```

Expected:

```text
DENIED
```

---

# 16. Observability

Recommended stack:

```text
Istio Ambient
     |
     +-- ztunnel metrics
     |
     +-- Waypoint metrics
     |
     +-- Prometheus
     |
     +-- Grafana
     |
     +-- Distributed tracing
```

Monitor:

```text
Request rate
Error rate
Latency
mTLS traffic
Authorization denials
HTTP status codes
Service dependencies
TCP connections
```

---

# 17. Security Model

The final security model is:

```text
                    DEFAULT DENY
                         |
                         v
                 +---------------+
                 | STRICT mTLS   |
                 +-------+-------+
                         |
                         v
                    ztunnel L4
                         |
                         v
                   Waypoint L7
                         |
                         v
               ServiceAccount identity
                         |
                         v
               Explicit ALLOW policy
                         |
                         v
                    Application
```

Security principles:

* Default deny
* Explicit service-to-service access
* Strong workload identity
* STRICT mTLS
* L4 enforcement at ztunnel
* L7 enforcement at waypoint
* Waypoint bypass protection
* Least-privilege communication

---

# 18. Deployment Order

Use this order:

```text
1. AWS VPC
       ↓
2. EKS
       ↓
3. IAM / OIDC
       ↓
4. Istio Base
       ↓
5. Istiod
       ↓
6. Istio CNI
       ↓
7. ztunnel
       ↓
8. RoboShop namespace
       ↓
9. ServiceAccounts
       ↓
10. RoboShop workloads
       ↓
11. Ambient enrollment
       ↓
12. Waypoint
       ↓
13. STRICT mTLS
       ↓
14. Default DENY
       ↓
15. L4 policies
       ↓
16. L7 policies
       ↓
17. Gateway API
       ↓
18. Observability
       ↓
19. Jenkins CI/CD
```

---

# 19. Troubleshooting

Check Istiod:

```bash
kubectl logs \
  -n istio-system \
  deploy/istiod
```

Check ztunnel:

```bash
kubectl logs \
  -n istio-system \
  -l app=ztunnel
```

Check workloads:

```bash
istioctl ztunnel-config workloads
```

Check authorization policies:

```bash
kubectl get authorizationpolicy \
  -n roboshop \
  -o yaml
```

Check services:

```bash
kubectl get svc -n roboshop
```

Check endpoints:

```bash
kubectl get endpoints -n roboshop
```

Check waypoint:

```bash
kubectl get pods -n roboshop
kubectl get gateway -n roboshop
```

---

# 20. Production Evolution

The project can be extended with:

```text
Istio Ambient
│
├── mTLS
├── Authorization
├── Gateway API
├── HTTP routing
├── Retries
├── Timeouts
├── Circuit breaking
├── Canary deployment
├── Telemetry
├── Distributed tracing
│
├── AWS
│   ├── EKS
│   ├── ALB/NLB
│   ├── IAM
│   ├── CloudWatch
│   └── VPC
│
└── CI/CD
    ├── Git
    ├── Jenkins
    ├── Terraform
    ├── Helm
    └── Kubernetes
```

---

# 21. Learning Outcomes

After completing this project, you will have hands-on experience with:

* AWS EKS
* Terraform
* Kubernetes
* Istio Ambient
* ztunnel
* Waypoint
* mTLS
* ServiceAccount identity
* AuthorizationPolicy
* L4 security
* L7 security
* Gateway API
* Service mesh observability
* Microservice security
* CI/CD
* Production-style cloud architecture

---

## License

This project is intended for learning, experimentation, and portfolio/interview demonstration.
