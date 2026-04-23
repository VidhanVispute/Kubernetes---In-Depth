# 📘 Chapter 4 — Namespaces & Pods

> This is where k8s starts feeling **real**. Pods are the smallest unit you'll deploy. Namespaces keep everything organized. You'll use both **every single day** as a DevOps engineer.

---

## 🔷 PART 1 — Namespaces

### What is a Namespace?

Think of your cluster as an **office building.**
Namespaces are the **floors** of that building.

```
┌─────────────────────────────────────────┐
│              K8s CLUSTER                │
│                                         │
│  ┌─────────────┐   ┌─────────────────┐  │
│  │  namespace: │   │   namespace:    │  │
│  │   default   │   │   production    │  │
│  │             │   │                 │  │
│  │ app-pod     │   │ app-pod         │  │
│  │ db-pod      │   │ db-pod          │  │
│  └─────────────┘   └─────────────────┘  │
│                                         │
│  ┌─────────────┐   ┌─────────────────┐  │
│  │  namespace: │   │   namespace:    │  │
│  │  dev-team   │   │   monitoring    │  │
│  └─────────────┘   └─────────────────┘  │
└─────────────────────────────────────────┘
```

Namespaces let you:
- **Isolate** environments (dev, staging, prod) on same cluster
- **Limit resources** per team (team A gets 4 CPU, team B gets 8 CPU)
- **Control access** (team A can't touch team B's namespace)
- **Avoid naming conflicts** (two teams can both have a pod called `app-pod`)

---

### 🔷 Default Namespaces in K8s

When you create a cluster, k8s gives you **4 namespaces out of the box:**

```bash
kubectl get namespaces

# NAME              STATUS
# default           Active   ← your resources go here if you don't specify
# kube-system       Active   ← k8s internal components (API server, etcd etc)
# kube-public       Active   ← readable by all, rarely used
# kube-node-lease   Active   ← node heartbeat tracking
```

> ⚠️ **Never deploy your apps into `kube-system`** — that's where k8s brain lives.

---

### 🛠️ Working With Namespaces

**Create a namespace:**
```bash
# Method 1 — command line
kubectl create namespace dev

# Method 2 — YAML (preferred in real jobs)
```
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```
```bash
kubectl apply -f namespace.yaml
```

---

**Deploy into a specific namespace:**
```bash
# Using flag
kubectl get pods --namespace=dev
kubectl get pods -n dev          # shorthand

# Apply resource into a namespace
kubectl apply -f pod.yaml -n dev

# Get resources across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A              # shorthand
```

---

**Set a default namespace (so you don't type -n every time):**
```bash
kubectl config set-context --current --namespace=dev

# Now all commands default to 'dev' namespace
kubectl get pods    # automatically looks in 'dev'
```

---

**Delete a namespace:**
```bash
# WARNING: This deletes EVERYTHING inside it
kubectl delete namespace dev
```

---

### 🔷 Real World Usage

```
Company XYZ K8s Cluster
│
├── namespace: dev          → developers test here freely
├── namespace: staging      → QA tests here before release
├── namespace: production   → live traffic, locked down
├── namespace: monitoring   → Prometheus, Grafana live here
└── namespace: logging      → ELK stack lives here
```

This is **exactly** how real companies structure their clusters. Remember this for interviews.

---

## 🔷 PART 2 — Pods

### What is a Pod?

> A Pod is the **smallest deployable unit** in Kubernetes.

**Not a container — a Pod.**

```
┌─────────────────────────────────┐
│              POD                │
│                                 │
│  ┌─────────────┐                │
│  │  Container  │  (usually 1)   │
│  │   (nginx)   │                │
│  └─────────────┘                │
│                                 │
│  Shared:                        │
│  • Network (same IP)            │
│  • Storage (volumes)            │
│  • localhost communication      │
└─────────────────────────────────┘
```

A pod **wraps** one or more containers and gives them:
- A **shared IP address**
- **Shared storage** (volumes)
- **Shared network namespace** (containers in same pod talk via localhost)

---

### 🔷 Why Pods and Not Just Containers?

Because sometimes **2 containers need to be tightly coupled:**

```
┌──────────────────────────────────────┐
│                POD                   │
│                                      │
│  ┌──────────────┐ ┌────────────────┐ │
│  │  Main App    │ │  Log Shipper   │ │
│  │  Container   │ │  (Sidecar)     │ │
│  │  (nginx)     │ │  (fluentd)     │ │
│  └──────────────┘ └────────────────┘ │
│                                      │
│  Share same volume → app writes logs │
│  fluentd reads & ships them          │
└──────────────────────────────────────┘
```

This is the **Sidecar pattern** — covered in a later chapter.

> **Rule of thumb:** 1 container per pod in most cases. Multi-container pods are for specific patterns only.

### 🚗 What is Sidecar Pattern (no nonsense)

Sidecar = an extra container running alongside your main app container inside the same Pod

👉 Same Pod
👉 Same network
👉 Same lifecycle

🧠 Simple definition (interview-ready)

“Sidecar pattern is a design where a helper container runs alongside the main application container in the same Pod to handle supporting concerns like logging, monitoring, or proxying.”

---

### 🛠️ Your First Pod — Hands On

**Method 1 — Imperative (quick, for testing):**
```bash
# Create a pod running nginx
kubectl run my-first-pod --image=nginx

# Check it
kubectl get pods

# NAME           READY   STATUS    RESTARTS   AGE
# my-first-pod   1/1     Running   0          10s
```

---

**Method 2 — Declarative YAML (always use this in real jobs):**

```yaml
# pod.yaml
apiVersion: v1          # k8s API version
kind: Pod               # type of resource
metadata:
  name: my-nginx-pod
  namespace: dev
  labels:               # key-value tags (very important — covered next)
    app: nginx
    env: dev
spec:
  containers:
    - name: nginx-container
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:         # minimum guaranteed resources
          memory: "64Mi"
          cpu: "250m"
        limits:           # maximum allowed resources
          memory: "128Mi"
          cpu: "500m"
```

```bash
# Apply it
kubectl apply -f pod.yaml

# Verify
kubectl get pods -n dev

# Detailed info
kubectl describe pod my-nginx-pod -n dev
```

---

### 🔷 Understanding the YAML Structure

Every k8s resource YAML has these **4 top-level fields:**

```yaml
apiVersion:   # which API version handles this resource
kind:         # what type of resource (Pod, Deployment, Service...)
metadata:     # name, namespace, labels, annotations
spec:         # the actual desired configuration
```

**This structure NEVER changes** — memorize it.

---

### 🔷 Pod Lifecycle — States You'll See

```
Pending   → Pod accepted, but container not started yet
            (waiting for image pull or node assignment)

Running   → At least one container is running

Succeeded → All containers completed successfully (for Jobs)

Failed    → All containers terminated, at least one failed

Unknown   → Can't get pod state (node communication issue)

CrashLoopBackOff → Container keeps crashing and restarting ⚠️
                   (most common error you'll debug)

ImagePullBackOff → Can't pull the container image ⚠️
                   (wrong image name or private registry issue)
```

---

### 🛠️ Most Used Pod Commands

```bash
# List pods
kubectl get pods
kubectl get pods -n dev
kubectl get pods -A                          # all namespaces
kubectl get pods -o wide                     # shows which node pod is on

# Detailed info (use this when debugging)
kubectl describe pod my-nginx-pod

# Logs (like docker logs)
kubectl logs my-nginx-pod
kubectl logs -f my-nginx-pod                 # follow/stream
kubectl logs my-nginx-pod -c nginx-container # specific container

# Shell into a pod (like docker exec)
kubectl exec -it my-nginx-pod -- /bin/bash
kubectl exec -it my-nginx-pod -- /bin/sh     # if bash not available

# Delete pod
kubectl delete pod my-nginx-pod
kubectl delete -f pod.yaml

# Watch pods in real time
kubectl get pods -w

# Get pod YAML (see what's actually stored in etcd)
kubectl get pod my-nginx-pod -o yaml
```

---

### 🔷 Port Forwarding — Access Pod Locally

Pods don't have external IPs by default. To access one locally:

```bash
# Forward localhost:8080 → pod port 80
kubectl port-forward pod/my-nginx-pod 8080:80

# Now open browser → http://localhost:8080
# You'll see nginx welcome page ✅
```

> This is great for **debugging** — not for production traffic.

---

### 🔷 Multi-Container Pod Example

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: app                    # main container
      image: nginx
      ports:
        - containerPort: 80

    - name: log-shipper            # sidecar container
      image: busybox
      command: ['sh', '-c', 'while true; do echo logging; sleep 5; done']
```

```bash
kubectl apply -f multi-container-pod.yaml

# Exec into specific container
kubectl exec -it multi-container-pod -c app -- /bin/bash
kubectl exec -it multi-container-pod -c log-shipper -- /bin/sh

# Logs of specific container
kubectl logs multi-container-pod -c log-shipper
```

---

### 🔷 Why You'll Never Create Pods Directly in Production

Pods by themselves are **not self-healing:**

```
Pod crashes
    ↓
It's GONE — nobody restarts it
    ↓
Your app is down 💀
```

That's why you use **Deployments** (next chapter) which wrap pods and ensure they're always running.

> **Rule:** In production, you never create naked pods. You always use Deployments, StatefulSets, or Jobs — which manage pods for you.

---

## ✅ Chapter 4 Summary

| Concept | Key Point |
|---|---|
| Namespace | Virtual cluster inside a cluster — isolates resources |
| Default namespaces | default, kube-system, kube-public, kube-node-lease |
| Pod | Smallest deployable unit, wraps 1+ containers |
| Pod YAML | apiVersion, kind, metadata, spec — always these 4 |
| Pod lifecycle | Pending → Running → Succeeded/Failed |
| CrashLoopBackOff | Container keeps crashing — most common error |
| Naked pods | Never use in production — use Deployments instead |

---

## 🧠 Interview Questions You Can Now Answer

1. **"What is a Pod in Kubernetes?"**
   → Smallest deployable unit. Wraps one or more containers, gives them a shared IP, storage, and network namespace.

2. **"What's the difference between a Pod and a container?"**
   → A container is a runtime process. A pod is a k8s abstraction that wraps containers and adds shared networking, storage, and lifecycle management.

3. **"What is CrashLoopBackOff?"**
   → The container keeps crashing and k8s keeps restarting it with exponential backoff. Debug with `kubectl logs` and `kubectl describe pod`.

4. **"Why use Namespaces?"**
   → Resource isolation, access control, resource quotas per team, avoid naming conflicts across environments.

