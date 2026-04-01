# 📘 Chapter 12 — HPA & VPA (Auto Scaling)

> Manual scaling is for demos. Production needs **automatic scaling** that reacts to real traffic. This chapter covers how k8s scales your apps intelligently — both horizontally and vertically.

---

## 🔷 The Problem First

```
Without autoscaling:

  Normal traffic:   3 pods running  → fine ✅
  Black Friday:     100x traffic    → 3 pods overwhelmed 💀
  Night time:       near-zero traffic → 3 pods wasting money 💸

Manual scaling options:
  → Wake up at 3am to scale up before traffic spike? ❌
  → Keep 50 pods running 24/7 just in case? ❌ (expensive)
  → Guess traffic patterns perfectly? ❌

With autoscaling:
  Traffic increases → k8s automatically adds pods ✅
  Traffic decreases → k8s automatically removes pods ✅
  You sleep peacefully 😴
```

---

## 🔷 3 Types of Scaling in K8s

```
┌──────────────────┬──────────────────────────────────────────┐
│  HPA             │  Horizontal Pod Autoscaler               │
│                  │  Adds/removes PODS                       │
│                  │  Scale OUT (more pods)                   │
├──────────────────┼──────────────────────────────────────────┤
│  VPA             │  Vertical Pod Autoscaler                 │
│                  │  Adjusts CPU/Memory of existing pods     │
│                  │  Scale UP (bigger pods)                  │
├──────────────────┼──────────────────────────────────────────┤
│  Cluster         │  Adds/removes NODES                      │
│  Autoscaler      │  When pods can't schedule → add node     │
│                  │  When nodes are underused → remove node  │
└──────────────────┴──────────────────────────────────────────┘
```

```
They work together:
  HPA adds pods → no room on existing nodes
  Cluster Autoscaler adds a new node
  Pods schedule on new node ✅

  Traffic drops → HPA removes pods
  Nodes become underutilized
  Cluster Autoscaler removes nodes
  Cloud bill drops ✅
```

---

## 🔷 PART 1 — HPA (Horizontal Pod Autoscaler)

### How HPA Works

```
┌──────────────────────────────────────────────────────────────┐
│                     HPA Control Loop                         │
│                                                              │
│  1. Metrics Server collects pod CPU/Memory every 15s         │
│                                                              │
│  2. HPA reads metrics from Metrics Server                    │
│                                                              │
│  3. HPA calculates desired replicas:                         │
│     desiredReplicas = ceil(currentReplicas ×                 │
│                       (currentMetric / targetMetric))        │
│                                                              │
│  4. HPA updates Deployment replicas                          │
│                                                              │
│  5. Deployment creates/removes pods                          │
│                                                              │
│  Repeat every 15 seconds ✅                                  │
└──────────────────────────────────────────────────────────────┘
```

---

### 🔷 Prerequisite — Metrics Server

HPA needs **Metrics Server** to get CPU/Memory data:

```bash
# Install Metrics Server (Minikube)
minikube addons enable metrics-server

# Install on other clusters
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify it's running
kubectl get pods -n kube-system | grep metrics-server
# metrics-server-7d9f9c7-xxxxx   1/1   Running ✅

# Test it works
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# minikube   250m         12%    1024Mi          26%

kubectl top pods -n dev
# NAME           CPU(cores)   MEMORY(bytes)
# nginx-pod      5m           8Mi
```

---

### 🛠️ HPA — CPU Based Scaling

**Step 1 — Deployment with resource requests (required for HPA):**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dev
spec:
  replicas: 2                    # starting replicas
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "200m"        # HPA uses this as baseline
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

**Step 2 — Create HPA:**

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app              # which deployment to scale

  minReplicas: 2               # never go below 2
  maxReplicas: 10              # never go above 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # scale when avg CPU > 70%
```

```bash
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa -n dev
# NAME          REFERENCE        TARGETS   MINPODS   MAXPODS   REPLICAS
# web-app-hpa   Deployment/web   25%/70%   2         10        2

# Describe for more detail
kubectl describe hpa web-app-hpa -n dev
```

---

### 🔷 HPA Scaling Math — Understand Deeply

```
Formula:
  desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))

Example:
  currentReplicas  = 2
  currentCPU       = 140% (each pod using 140% of requested 200m)
  targetCPU        = 70%

  desiredReplicas = ceil(2 × (140 / 70))
                 = ceil(2 × 2)
                 = ceil(4)
                 = 4 pods

  HPA scales from 2 → 4 pods ✅

Scale down example:
  currentReplicas = 4
  currentCPU      = 20%
  targetCPU       = 70%

  desiredReplicas = ceil(4 × (20 / 70))
                 = ceil(4 × 0.28)
                 = ceil(1.14)
                 = 2 pods  (but min is 2 so stays at 2)
```

---

### 🛠️ HPA — Memory Based Scaling

```yaml
# memory-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa-memory
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # scale when memory > 80%
```

> ⚠️ **Memory-based HPA gotcha:** Memory doesn't go down when you add pods (unlike CPU). A memory-hungry app needs more RAM per pod — not more pods. VPA is better for memory issues.

---

### 🛠️ HPA — Multiple Metrics

```yaml
# multi-metric-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # CPU metric
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Memory metric
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # Custom metric — requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"        # scale if > 1000 req/s per pod

# HPA scales when ANY metric exceeds target
# Takes the MAXIMUM desired replicas across all metrics
```

---

### 🔷 HPA Scaling Behavior — Control Speed

```yaml
# hpa-with-behavior.yaml
spec:
  minReplicas: 2
  maxReplicas: 20
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up immediately
      policies:
        - type: Percent
          value: 100                   # can double pods per period
          periodSeconds: 15
        - type: Pods
          value: 4                     # OR add max 4 pods per period
          periodSeconds: 15
      selectPolicy: Max                # use whichever adds more pods

    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5min before scaling down
      policies:
        - type: Percent
          value: 10                    # remove max 10% of pods per period
          periodSeconds: 60
      selectPolicy: Min                # use whichever removes fewer pods
```

```
Why stabilizationWindow for scale down?

  Traffic spike → HPA scales to 20 pods
  Spike ends    → HPA wants to scale down immediately
  But spike might come back in 2 minutes!
  Without window → yo-yo scaling (up-down-up-down)
  With 5min window → waits to confirm traffic really dropped
  Saves thrashing, more stable ✅
```

---

### 🛠️ Load Test — Watch HPA in Action

```bash
# Terminal 1 — watch HPA
kubectl get hpa web-app-hpa -n dev -w

# Terminal 2 — generate load
kubectl run load-test --image=busybox \
  --restart=Never -n dev -- \
  /bin/sh -c "while true; do wget -q -O- http://web-app-service; done"

# Watch Terminal 1:
# NAME          TARGETS    MINPODS   MAXPODS   REPLICAS
# web-app-hpa   25%/70%    2         10        2
# web-app-hpa   82%/70%    2         10        2        ← going up
# web-app-hpa   95%/70%    2         10        4        ← scaled to 4!
# web-app-hpa   88%/70%    2         10        4
# web-app-hpa   71%/70%    2         10        6        ← scaled to 6!
# web-app-hpa   65%/70%    2         10        6        ← stabilizing

# Stop load test
kubectl delete pod load-test -n dev

# After 5 min stabilization window:
# web-app-hpa   20%/70%    2         10        4        ← scaling down
# web-app-hpa   15%/70%    2         10        2        ← back to min ✅
```

---

## 🔷 PART 2 — VPA (Vertical Pod Autoscaler)

### What is VPA?

```
HPA = more pods (scale out)
VPA = bigger pods (scale up)

VPA automatically adjusts:
  CPU requests/limits
  Memory requests/limits

Based on actual usage history
```

```
Use case:
  You set requests: cpu=200m, memory=128Mi
  App actually uses: cpu=800m, memory=512Mi

  Without VPA:
    → App constantly throttled (slow)
    → Scheduler places it on wrong-sized nodes
    → Manual tuning nightmare

  With VPA:
    → Analyzes usage over time
    → Recommends/sets: cpu=900m, memory=600Mi
    → Right-sized automatically ✅
```

---

### 🔷 VPA Modes

```
┌──────────────┬──────────────────────────────────────────────┐
│  Off         │  Only provides recommendations               │
│              │  Does NOT change anything                    │
│              │  Use to: learn what values to set            │
├──────────────┼──────────────────────────────────────────────┤
│  Initial     │  Sets resources only at pod CREATION         │
│              │  Does NOT change running pods                │
│              │  Safer than Auto mode                        │
├──────────────┼──────────────────────────────────────────────┤
│  Auto        │  Changes resources on running pods           │
│              │  Requires pod restart (evicts + recreates)   │
│              │  Most powerful but disruptive                │
│              │                                              │
│  Recreate    │  Same as Auto                                │
└──────────────┴──────────────────────────────────────────────┘
```

---

### 🛠️ Install VPA

```bash
# Clone VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh

# Verify VPA pods running
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-xxx   1/1   Running ✅
# vpa-recommender-xxx            1/1   Running ✅
# vpa-updater-xxx                1/1   Running ✅
```

---

### 🛠️ VPA YAML

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: dev
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app              # which deployment to target

  updatePolicy:
    updateMode: "Auto"         # Off / Initial / Auto

  resourcePolicy:
    containerPolicies:
      - containerName: web     # target specific container
        minAllowed:
          cpu: "100m"          # never set below this
          memory: "64Mi"
        maxAllowed:
          cpu: "2000m"         # never set above this
          memory: "2Gi"
        controlledResources:
          - cpu
          - memory
```

```bash
kubectl apply -f vpa.yaml

# Check VPA recommendations
kubectl describe vpa web-app-vpa -n dev

# Output:
# Recommendation:
#   Container Recommendations:
#     Container Name: web
#     Lower Bound:
#       Cpu:     200m
#       Memory:  128Mi
#     Target:                  ← what VPA will set
#       Cpu:     850m
#       Memory:  512Mi
#     Upper Bound:
#       Cpu:     1200m
#       Memory:  768Mi
#     Uncapped Target:         ← without min/max constraints
#       Cpu:     920m
#       Memory:  530Mi
```

---

### 🔷 VPA Off Mode — Just Get Recommendations

```yaml
# vpa-recommendation-only.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: dev
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Off"           # recommendations only — no changes
```

```bash
# Use this first to understand your app's real resource needs
# Then manually update your deployment with recommended values
# Then switch to Auto if you trust the recommendations

kubectl describe vpa web-app-vpa -n dev
# Check Target values → copy to your deployment YAML
```

---

## 🔷 PART 3 — HPA vs VPA

```
┌───────────────────┬──────────────────┬──────────────────────┐
│                   │       HPA        │        VPA           │
├───────────────────┼──────────────────┼──────────────────────┤
│ What it scales    │ Pod COUNT        │ Pod SIZE (resources) │
│ How               │ Add/remove pods  │ Resize requests/lims │
│ Best for          │ Stateless apps   │ Stateful apps        │
│                   │ Web, API, workers│ DB, ML models        │
│ Metrics used      │ CPU, memory,     │ Historical CPU       │
│                   │ custom metrics   │ and memory usage     │
│ Pod restart       │ No               │ Yes (in Auto mode)   │
│ Works with        │ Deployments,     │ Deployments,         │
│                   │ StatefulSets     │ StatefulSets, etc    │
│ Conflict risk     │ Don't use both   │ Don't use both       │
│                   │ for CPU/memory   │ for CPU/memory       │
└───────────────────┴──────────────────┴──────────────────────┘
```

---

### 🔷 Using HPA + VPA Together (Safe Pattern)

```yaml
# Safe combination:
# VPA manages memory (vertical)
# HPA manages CPU (horizontal)

# VPA — only control memory
spec:
  resourcePolicy:
    containerPolicies:
      - containerName: web
        controlledResources:
          - memory              # VPA only touches memory
                                # NOT cpu

# HPA — only scale on CPU
spec:
  metrics:
    - type: Resource
      resource:
        name: cpu               # HPA only uses CPU
                                # NOT memory
```

> ⚠️ **Never let both HPA and VPA control the same resource (e.g. both controlling CPU).** They'll fight each other and cause instability.

---

## 🔷 PART 4 — Cluster Autoscaler

### What is Cluster Autoscaler?

```
HPA/VPA scale pods
Cluster Autoscaler scales NODES

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Pods Pending (can't schedule — no room)                │
│       ↓                                                 │
│  Cluster Autoscaler detects pending pods                │
│       ↓                                                 │
│  Requests new node from cloud provider (AWS/GCP/Azure)  │
│       ↓                                                 │
│  Node joins cluster                                     │
│       ↓                                                 │
│  Pending pods schedule on new node ✅                   │
│                                                         │
│  Nodes underutilized (< 50% for 10+ min)                │
│       ↓                                                 │
│  Cluster Autoscaler drains + removes node               │
│       ↓                                                 │
│  Pods rescheduled on other nodes                        │
│       ↓                                                 │
│  Cloud bill reduced ✅                                   │
└─────────────────────────────────────────────────────────┘
```

---

### 🛠️ Cluster Autoscaler on AWS EKS

```bash
# Step 1: Create node group with autoscaling enabled
eksctl create nodegroup \
  --cluster my-cluster \
  --name managed-ng \
  --instance-type t3.medium \
  --nodes-min 2 \
  --nodes-max 10 \
  --asg-access          # grants autoscaler IAM access

# Step 2: Install Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Step 3: Annotate with cluster name
kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT:role/ClusterAutoscalerRole

# Step 4: Set cluster name
kubectl set env deployment cluster-autoscaler \
  -n kube-system \
  CLUSTER_NAME=my-cluster

# Watch it work
kubectl logs -f deployment/cluster-autoscaler \
  -n kube-system | grep -i scale
```

---

## 🔷 Full Auto-Scaling Architecture — Production

```
                    Internet Traffic
                          │
                  [Load Balancer]
                          │
                    [Ingress]
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
         [Pod-1]                 [Pod-2]
              │         HPA           │
              └────── scales ─────────┘
                    pods 2→10
                          │
              No room on existing nodes
                          │
                 Cluster Autoscaler
                    adds Node-4
                          │
              Pods schedule on Node-4 ✅

              Traffic drops at night
                          │
                   HPA scales down
                    pods 10→2
                          │
              Nodes underutilized
                          │
                 Cluster Autoscaler
                    removes Node-4
                          │
              Cloud bill drops ✅
```

---

## ✅ Chapter 12 Summary

| Concept | Key Point |
|---|---|
| HPA | Scales pod COUNT based on metrics |
| VPA | Scales pod SIZE (CPU/memory requests) |
| Cluster Autoscaler | Scales node COUNT when pods can't schedule |
| Metrics Server | Required for HPA — collects CPU/memory data |
| minReplicas / maxReplicas | Hard bounds on HPA scaling |
| stabilizationWindowSeconds | Prevents yo-yo scaling on scale down |
| VPA Off mode | Recommendations only — safe starting point |
| VPA Auto mode | Automatically resizes pods — requires restart |
| HPA + VPA together | Use VPA for memory, HPA for CPU — don't overlap |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is HPA and how does it work?"**
   → HPA monitors metrics via Metrics Server, calculates desired pod count using the formula `ceil(currentReplicas × currentMetric/targetMetric)`, and updates the Deployment replica count automatically.

2. **"What's the difference between HPA and VPA?"**
   → HPA adds/removes pods horizontally — best for stateless apps with variable traffic. VPA adjusts CPU and memory of existing pods vertically — best for right-sizing stateful apps or workloads that don't parallelize well.

3. **"Can you use HPA and VPA together?"**
   → Yes, safely if they control different resources. VPA manages memory, HPA manages CPU. Never let both control the same resource — they'll conflict and cause instability.

4. **"What is the Cluster Autoscaler?"**
   → Scales the number of nodes in the cluster. Adds nodes when pods are Pending due to insufficient resources. Removes underutilized nodes to save cost. Works with cloud provider node groups (ASGs on AWS).

5. **"What happens if HPA wants 10 pods but no node has capacity?"**
   → Pods go Pending. Cluster Autoscaler detects Pending pods, provisions a new node from the cloud provider, and the pods schedule on it. All three autoscalers work together.

---

Ready for **Chapter 13 → RBAC** — Role Based Access Control — how to secure your cluster so the right people and services have exactly the right permissions? One of the most important production topics. Say **"next"** 🚀
