# 📘 Chapter 8 — Services & Networking

> Without Services, pods are islands. They have IPs that change every restart and can't be reliably reached. Services solve this completely. This chapter is **heavily tested in interviews.**

---

## 🔷 The Problem Services Solve

```
Pod IP addresses are:
  → Assigned randomly when pod starts
  → Changed every time pod restarts
  → Different after rescheduling to another node

So how does frontend pod reliably talk to backend pod?
How does external traffic reach your app?

Answer: Services
```

```
WITHOUT Service:
  Frontend → calls 10.244.1.15:8080 (backend pod IP)
  Backend pod crashes & restarts → new IP: 10.244.2.31
  Frontend → still calling 10.244.1.15 → CONNECTION REFUSED 💀

WITH Service:
  Frontend → calls backend-service:8080 (stable DNS name)
  Service → always knows current pod IPs via label selector
  Backend pod restarts → Service updates automatically ✅
```

---

## 🔷 How Services Find Pods

Services use **label selectors** — same concept from Chapter 5:

```
Service:
  selector:
    app: backend          ← find all pods with this label

Pods:
  Pod-1 labels: app=backend  ← matched ✅
  Pod-2 labels: app=backend  ← matched ✅
  Pod-3 labels: app=frontend ← NOT matched ❌

Service load balances traffic across Pod-1 and Pod-2
```

When pods are added/removed, the Service's **Endpoints** object
updates automatically. Zero manual work.

```bash
# See what pods a service is routing to
kubectl get endpoints backend-service -n dev

# NAME              ENDPOINTS
# backend-service   10.244.1.15:8080,10.244.2.31:8080  ✅
```

---

## 🔷 The 4 Service Types

```
┌─────────────────┬────────────────────────────────────────────┐
│  ClusterIP      │ Internal only — pod to pod communication   │
│  NodePort       │ Exposes on every node's IP + port          │
│  LoadBalancer   │ Cloud load balancer — production external  │
│  ExternalName   │ Maps service to external DNS name          │
└─────────────────┴────────────────────────────────────────────┘
```

Let's go deep into each one.

---

## 🔷 PART 1 — ClusterIP (Default)

### What is ClusterIP?

```
┌──────────────────────────────────────────────┐
│               CLUSTER                        │
│                                              │
│  [Frontend Pod] ──▶ [ClusterIP Service]      │
│                           │                  │
│                    ┌──────┴──────┐           │
│                    ▼             ▼           │
│              [Backend Pod-1] [Backend Pod-2] │
│                                              │
│  ❌ NOT accessible from outside cluster      │
└──────────────────────────────────────────────┘
```

- Only accessible **within the cluster**
- Gets a stable **virtual IP** (the ClusterIP)
- Load balances across matching pods
- **Default type** if you don't specify

---

### 🛠️ ClusterIP YAML

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: dev
spec:
  type: ClusterIP          # default — can omit this line
  selector:
    app: backend           # route to pods with this label
  ports:
    - name: http
      protocol: TCP
      port: 80             # port the SERVICE listens on
      targetPort: 8080     # port the POD/container listens on
```

```bash
kubectl apply -f clusterip-service.yaml

kubectl get service -n dev
# NAME              TYPE        CLUSTER-IP      PORT(S)
# backend-service   ClusterIP   10.96.45.123    80/TCP

# Test from another pod inside cluster
kubectl exec -it frontend-pod -n dev -- curl backend-service:80
# OR using full DNS name:
kubectl exec -it frontend-pod -n dev -- \
  curl backend-service.dev.svc.cluster.local:80
```

---

### 🔷 K8s DNS — How Service Discovery Works

Every Service gets a DNS name automatically:

```
Format:
  <service-name>.<namespace>.svc.cluster.local

Examples:
  backend-service.dev.svc.cluster.local
  mysql.dev.svc.cluster.local
  redis.production.svc.cluster.local

Shorthand (within same namespace):
  backend-service          ← just the service name works!
  mysql:3306

Cross-namespace:
  backend-service.dev      ← service.namespace
```

> This DNS-based service discovery is how microservices find each other in k8s. No hardcoded IPs anywhere.

---

## 🔷 PART 2 — NodePort

### What is NodePort?

```
┌───────────────────────────────────────────────┐
│                  CLUSTER                      │
│                                               │
│  Node-1 (IP: 54.12.34.56)                    │
│  ┌────────────────────────────────────┐       │
│  │  NodePort: 30080                   │       │
│  │       │                            │       │
│  │  [ClusterIP Service :80]           │       │
│  │       │                            │       │
│  │  [Pod-1 :8080]  [Pod-2 :8080]      │       │
│  └────────────────────────────────────┘       │
│                                               │
│  Node-2 (IP: 54.12.34.57)                    │
│  ┌────────────────────────────────────┐       │
│  │  NodePort: 30080 (same port!)      │       │
│  └────────────────────────────────────┘       │
└───────────────────────────────────────────────┘

External access:
  http://54.12.34.56:30080  ✅
  http://54.12.34.57:30080  ✅  (any node works!)
```

- Opens a **port on every node** in the cluster (30000-32767 range)
- External traffic hits any node IP + NodePort
- k8s routes it to the right pod automatically
- **Not for production** — exposes node IPs directly, no SSL termination

---

### 🛠️ NodePort YAML

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dev
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80           # ClusterIP port (internal)
      targetPort: 3000   # container port
      nodePort: 30080    # external port on node (30000-32767)
                         # omit this to let k8s auto-assign
```

```bash
kubectl apply -f nodeport-service.yaml

kubectl get service -n dev
# NAME               TYPE       CLUSTER-IP     PORT(S)
# frontend-service   NodePort   10.96.78.90    80:30080/TCP

# Access it (minikube example)
minikube ip           # get node IP e.g. 192.168.49.2
curl 192.168.49.2:30080

# Minikube shortcut
minikube service frontend-service -n dev
```

---

## 🔷 PART 3 — LoadBalancer

### What is a LoadBalancer Service?

```
Internet
    │
    ▼
[AWS ALB/NLB]  ← automatically created by k8s
    │
    ▼
┌─────────────────────────────────────────┐
│              CLUSTER                    │
│                                         │
│  [LoadBalancer Service]                 │
│         │                               │
│  ┌──────┴──────┐                        │
│  ▼             ▼                        │
│ [Pod-1]      [Pod-2]                    │
└─────────────────────────────────────────┘

External access:
  http://abc123.elb.amazonaws.com  ✅
```

- Creates a **real cloud load balancer** (AWS ALB/NLB, GCP LB, Azure LB)
- Gets an **external IP/DNS** automatically
- Best for production external traffic
- **Each LoadBalancer service = 1 cloud LB = costs money** ⚠️

---

### 🛠️ LoadBalancer YAML

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80           # external port
      targetPort: 3000   # container port
```

```bash
kubectl apply -f loadbalancer-service.yaml

# Watch for external IP/DNS (takes 1-2 min on AWS)
kubectl get service app-service -n production -w

# NAME          TYPE          CLUSTER-IP    EXTERNAL-IP
# app-service   LoadBalancer  10.96.45.12   <pending>
# app-service   LoadBalancer  10.96.45.12   abc123.elb.amazonaws.com ✅

# Access it
curl http://abc123.elb.amazonaws.com
```

---

## 🔷 PART 4 — Ingress

### The Problem With LoadBalancer Services

```
You have 5 microservices:
  /api/users    → users-service
  /api/orders   → orders-service
  /api/payments → payments-service
  /             → frontend-service
  /admin        → admin-service

With LoadBalancer type:
  5 services = 5 cloud LBs = 5 external IPs = 💰💰💰
  No SSL termination in one place
  No path-based routing
  No host-based routing

With Ingress:
  1 cloud LB → Ingress Controller → routes to all services
  SSL termination in one place ✅
  Path-based routing ✅
  Host-based routing ✅
  Cost efficient ✅
```

---

### 🔷 Ingress Architecture

```
Internet
    │
    ▼
[1 Cloud Load Balancer]
    │
    ▼
[Ingress Controller]  ← actual routing engine (nginx, traefik etc)
    │
    ├── /api/users    → users-service:80
    ├── /api/orders   → orders-service:80
    ├── /api/payments → payments-service:80
    └── /             → frontend-service:80
```

Two things needed:
1. **Ingress Controller** — the actual proxy (nginx, traefik, AWS ALB controller)
2. **Ingress Resource** — your routing rules (YAML you write)

---

### 🛠️ Install Ingress Controller (Minikube)

```bash
# Minikube — enable nginx ingress addon
minikube addons enable ingress

# Verify controller is running
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-7d98fb6bb4-xxxxx   1/1     Running ✅
```

```bash
# AWS EKS — install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### 🛠️ Ingress Resource YAML

**Path-based routing:**

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: dev
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx          # which ingress controller to use
  rules:
    - host: myapp.example.com      # domain name
      http:
        paths:
          - path: /                # root path → frontend
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          - path: /api/users       # path → users service
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80

          - path: /api/orders      # path → orders service
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
```

---

**Host-based routing (multiple domains):**

```yaml
# multi-host-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com         # main app
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.example.com         # API subdomain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80

    - host: admin.example.com       # admin panel
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

---

**TLS/HTTPS with Ingress:**

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: dev
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # auto SSL
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls-secret   # cert stored here as k8s Secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml

kubectl get ingress -n dev
# NAME          CLASS   HOSTS               ADDRESS          PORTS
# app-ingress   nginx   myapp.example.com   192.168.49.2     80,443

# Describe to see routing rules
kubectl describe ingress app-ingress -n dev
```

---

## 🔷 PART 5 — Quick Project: Django Notes App

> Let's tie it all together with a real app deployment

```yaml
# django-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-notes
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-notes
  template:
    metadata:
      labels:
        app: django-notes
    spec:
      containers:
        - name: django
          image: your-dockerhub/django-notes:latest
          ports:
            - containerPort: 8000
          env:
            - name: DEBUG
              value: "False"
            - name: DB_HOST
              value: "mysql-service"    # talks to mysql via service name
---
apiVersion: v1
kind: Service
metadata:
  name: django-notes-service
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: django-notes
  ports:
    - port: 80
      targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-notes-ingress
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
    - host: notes.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-notes-service
                port:
                  number: 80
```

```bash
kubectl apply -f django-deployment.yaml

# Full stack running:
# Ingress → Service → Deployment → Pods → Django App ✅
```

---

## 🔷 Service Types — Full Comparison

```
┌───────────────┬──────────────┬───────────────────────────────────┐
│  Type         │  Access      │  Use case                         │
├───────────────┼──────────────┼───────────────────────────────────┤
│ ClusterIP     │ Internal     │ Pod-to-pod, microservice comms    │
│ NodePort      │ Node IP+Port │ Dev/testing, simple external      │
│ LoadBalancer  │ Cloud LB     │ Single service production         │
│ Ingress       │ Cloud LB +   │ Multi-service production,         │
│               │ smart routing│ path/host routing, SSL            │
└───────────────┴──────────────┴───────────────────────────────────┘
```

---

## ✅ Chapter 8 Summary

| Concept | Key Point |
|---|---|
| Service | Stable endpoint for pods — survives pod restarts |
| ClusterIP | Internal only — default type — pod to pod |
| NodePort | Exposes on node IP — dev/test only |
| LoadBalancer | Cloud LB — production for single service |
| Ingress | Smart HTTP router — 1 LB for all services |
| Ingress Controller | The actual proxy engine (nginx, traefik) |
| Ingress Resource | Your routing rules in YAML |
| DNS | `service.namespace.svc.cluster.local` |
| Endpoints | Auto-updated list of pod IPs behind a service |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is a Kubernetes Service and why do we need it?"**
   → Pods get random IPs that change on restart. Service provides a stable endpoint with a fixed DNS name and IP, load balancing across all matching pods automatically.

2. **"What's the difference between ClusterIP, NodePort and LoadBalancer?"**
   → ClusterIP is internal only. NodePort exposes a port on every node for external access. LoadBalancer creates a cloud load balancer with a public IP. Each builds on the previous.

3. **"What is an Ingress and how is it different from a LoadBalancer service?"**
   → LoadBalancer creates one cloud LB per service — expensive for many services. Ingress uses one LB with an Ingress Controller to route traffic to multiple services based on path or hostname — cost-efficient and supports SSL termination.

4. **"What is an Ingress Controller?"**
   → The actual implementation that processes Ingress rules. Popular ones are nginx, traefik, and AWS ALB Controller. Ingress resource = rules. Ingress Controller = the engine that enforces them.

5. **"How do pods in different namespaces communicate?"**
   → Using the full DNS name: `service-name.namespace.svc.cluster.local`. Within the same namespace just the service name works.

---

Ready for **Chapter 9 → ConfigMaps & Secrets** — how to manage configuration and sensitive data in k8s the right way? Say **"next"** 🚀
