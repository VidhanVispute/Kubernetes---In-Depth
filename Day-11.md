# 📘 Chapter 11 — Taints, Tolerations & Node Affinity

> This chapter is about **control**. In production you have different types of nodes — GPU nodes, spot instances, high-memory nodes, nodes for specific teams. This is how you control which pods land where.

---

## 🔷 The Problem First

```
Production cluster has mixed nodes:

  Node-1: GPU node     (expensive, $$$)  → only ML workloads
  Node-2: Spot node    (cheap, unstable) → only batch jobs
  Node-3: High-memory  (16GB RAM)        → only databases
  Node-4: General      (standard)        → everything else

Without controls:
  Scheduler places pods randomly
  ML pod lands on general node → no GPU → crashes
  Database lands on spot node  → spot terminated → data risk
  Random pods fill GPU node    → ML team can't schedule

With Taints/Tolerations + Node Affinity:
  Each node only accepts the right pods ✅
```

---

## 🔷 Two Mechanisms — Know the Difference

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  TAINTS + TOLERATIONS  →  Node says "STAY AWAY"            │
│  (push mechanism)         Pod says "I can tolerate that"   │
│                                                             │
│  NODE AFFINITY         →  Pod says "I WANT to go there"    │
│  (pull mechanism)         Based on node labels             │
│                                                             │
│  Use both together for precise control ✅                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 PART 1 — Taints & Tolerations

### What is a Taint?

A taint is applied to a **node**.
It says: **"Repel all pods unless they tolerate this taint."**

```
Node with taint:
  gpu=true:NoSchedule

Effect on scheduler:
  Regular pod    → ❌ cannot schedule on this node
  Pod with toleration for gpu=true → ✅ can schedule here
```

---

### 🔷 Taint Structure

```
kubectl taint nodes <node-name> key=value:effect

Three parts:
  key=value  →  the taint identifier
               e.g. gpu=true, team=ml, env=prod

  effect     →  what happens to intolerant pods
               NoSchedule    → don't schedule here (hard rule)
               PreferNoSchedule → try not to, but ok if needed
               NoExecute     → evict existing pods + no new ones
```

---

### 🛠️ Adding Taints to Nodes

```bash
# Taint a node — only GPU workloads allowed
kubectl taint nodes node-1 gpu=true:NoSchedule

# Taint for spot instances
kubectl taint nodes node-2 spot=true:NoSchedule

# Taint with NoExecute — evicts existing pods too
kubectl taint nodes node-3 maintenance=true:NoExecute

# View taints on a node
kubectl describe node node-1 | grep Taints
# Taints: gpu=true:NoSchedule

# Remove a taint (add minus at end)
kubectl taint nodes node-1 gpu=true:NoSchedule-
```

---

### 🔷 Taint Effects Deep Dive

```
NoSchedule:
  → New pods without toleration: NOT scheduled here
  → Existing pods already running: NOT evicted (stay)
  → Use for: dedicated nodes going forward

PreferNoSchedule:
  → Scheduler TRIES to avoid this node
  → If no other option: pod CAN land here
  → Soft rule — not guaranteed
  → Use for: soft preferences

NoExecute:
  → New pods without toleration: NOT scheduled
  → Existing pods without toleration: EVICTED immediately
  → Existing pods WITH toleration: can set tolerationSeconds
  → Use for: node maintenance, draining nodes
```

---

### What is a Toleration?

A toleration is applied to a **pod**.
It says: **"I can tolerate this taint — schedule me there."**

```yaml
# pod-with-toleration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training-pod
  namespace: dev
spec:
  tolerations:
    - key: "gpu"               # must match taint key
      operator: "Equal"        # Equal or Exists
      value: "true"            # must match taint value
      effect: "NoSchedule"     # must match taint effect

  containers:
    - name: ml-trainer
      image: tensorflow/tensorflow:latest-gpu
      resources:
        limits:
          nvidia.com/gpu: 1    # request 1 GPU
```

---

### 🔷 Toleration Operators

```yaml
# Equal — key AND value must match exactly
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"

# Exists — only key needs to match (any value)
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"

# Tolerate ALL taints on a node (empty key)
tolerations:
  - operator: "Exists"      # matches any taint
# Use for: system pods that must run everywhere
# (kube-proxy, fluentd DaemonSets use this)
```

---

### 🔷 NoExecute with tolerationSeconds

```yaml
# pod that tolerates maintenance taint — but only for 1 hour
tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 3600   # stay for 1 hour then evict

# Use case:
# Node going into maintenance
# Critical pods get graceful 1-hour window to finish work
# After 1 hour → evicted and rescheduled elsewhere
```

---

### 🔷 Real World Taint Scenarios

**Scenario 1 — Dedicated GPU nodes:**
```bash
# Admin taints GPU nodes
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
kubectl taint nodes gpu-node-2 gpu=true:NoSchedule

# Only ML pods with this toleration land on GPU nodes
# All other pods stay off GPU nodes automatically ✅
```

**Scenario 2 — Spot instance nodes:**
```bash
# AWS adds this taint automatically to spot nodes
kubectl taint nodes spot-node-1 \
  node.kubernetes.io/spot=true:NoSchedule

# Only batch jobs with spot toleration schedule here
# Regular production pods never land on unstable spot nodes ✅
```

**Scenario 3 — Node maintenance:**
```bash
# Node going down for maintenance
kubectl taint nodes worker-3 maintenance=true:NoExecute

# All pods evicted immediately
# Rescheduled on healthy nodes
# Safer than: kubectl drain (which also cordons) ✅
```

---

## 🔷 PART 2 — Node Affinity

### What is Node Affinity?

While taints repel pods, **Node Affinity attracts pods to specific nodes** based on node labels.

```
Taints/Tolerations  →  "Keep pods OFF this node"  (push)
Node Affinity       →  "Put pods ON this node"    (pull)
```

---

### 🔷 Step 1 — Label Your Nodes

```bash
# Label nodes with their characteristics
kubectl label nodes node-1 gpu=true
kubectl label nodes node-1 instance-type=p3.2xlarge

kubectl label nodes node-2 disk=ssd
kubectl label nodes node-2 zone=us-east-1a

kubectl label nodes node-3 team=data-engineering
kubectl label nodes node-3 memory=high

# View node labels
kubectl get nodes --show-labels
kubectl describe node node-1 | grep Labels
```

---

### 🔷 Node Affinity Types

```
requiredDuringSchedulingIgnoredDuringExecution
  → HARD rule
  → Pod MUST land on matching node
  → If no matching node → pod stays Pending
  → "Required" = must have this

preferredDuringSchedulingIgnoredDuringExecution
  → SOFT rule
  → Scheduler PREFERS matching node
  → If no matching node → schedules elsewhere
  → "Preferred" = nice to have
```

> "IgnoredDuringExecution" means: if node labels change after pod is running, pod is NOT evicted. (There's a future `RequiredDuringExecution` type coming but not stable yet.)

---

### 🛠️ Node Affinity YAML

**Required (hard) affinity:**

```yaml
# required-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: dev
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu              # node must have this label
                operator: In
                values:
                  - "true"           # label value must be "true"

  containers:
    - name: ml-app
      image: tensorflow/tensorflow:gpu
```

---

**Preferred (soft) affinity:**

```yaml
# preferred-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: dev
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80                  # higher weight = stronger preference
          preference:
            matchExpressions:
              - key: disk
                operator: In
                values:
                  - ssd               # prefer SSD nodes

        - weight: 20                  # lower weight = weaker preference
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-east-1a        # prefer this AZ

  containers:
    - name: web
      image: nginx:1.25
```

---

**Both required AND preferred:**

```yaml
spec:
  affinity:
    nodeAffinity:

      # MUST be on a node with gpu=true
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: In
                values: ["true"]

      # PREFER nodes in us-east-1a
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a"]
```

---

### 🔷 Affinity Operators

```
In          → label value is IN the list
              key: zone, values: [us-east-1a, us-east-1b]

NotIn       → label value NOT in the list
              key: env, values: [test, dev]  ← avoid test/dev nodes

Exists      → label key exists (any value)
              key: gpu  ← node has any gpu label

DoesNotExist → label key does NOT exist
               key: tainted  ← node has no 'tainted' label

Gt          → label value greater than number
              key: memory-gb, values: ["16"]  ← 16GB+ nodes

Lt          → label value less than number
              key: cpu-cores, values: ["8"]   ← less than 8 cores
```

---

## 🔷 PART 3 — Pod Affinity & Anti-Affinity

### Pod Affinity — Place pods NEAR each other

```
Use case:
  Frontend pod and backend pod
  → should be on same node or same zone
  → to reduce network latency
```

```yaml
# pod-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-pod
  namespace: dev
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - backend          # schedule near pods with app=backend
          topologyKey: "kubernetes.io/hostname"
          # topologyKey = what "near" means:
          # hostname → same node
          # zone     → same availability zone
          # region   → same region

  containers:
    - name: frontend
      image: frontend:latest
```

---

### Pod Anti-Affinity — Spread pods AWAY from each other

```
Use case:
  3 replicas of same app
  → spread across different nodes
  → if one node dies, not all replicas die
```

```yaml
# pod-anti-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - web          # don't schedule near other web pods
              topologyKey: "kubernetes.io/hostname"
              # Each replica MUST be on different node ✅

      containers:
        - name: web
          image: nginx:1.25
```

```
Result:
  Pod-1 → Node-1   ✅
  Pod-2 → Node-2   ✅
  Pod-3 → Node-3   ✅

  Node-1 dies → Pod-1 lost
  But Pod-2 and Pod-3 still running on other nodes
  App still available ✅
```

---

## 🔷 PART 4 — Taints + Affinity Together (Production Pattern)

### Dedicated Node Pool Pattern

```
Goal:
  GPU nodes → ONLY ML workloads
  No other pods can land there
  ML pods ALWAYS land there

Solution:
  Taint GPU nodes     → repels everyone except ML pods
  Node Affinity on ML → attracts ML pods to GPU nodes
```

```bash
# Step 1: Label GPU nodes
kubectl label nodes gpu-node-1 workload=ml
kubectl label nodes gpu-node-2 workload=ml

# Step 2: Taint GPU nodes
kubectl taint nodes gpu-node-1 dedicated=ml:NoSchedule
kubectl taint nodes gpu-node-2 dedicated=ml:NoSchedule
```

```yaml
# Step 3: ML pod has BOTH toleration AND affinity
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  # Toleration = can land on tainted GPU node
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "ml"
      effect: "NoSchedule"

  # Affinity = PREFER to land on GPU node
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: workload
                operator: In
                values: ["ml"]

  containers:
    - name: trainer
      image: tensorflow/tensorflow:latest-gpu
```

```
Result:
  GPU nodes: ONLY accept ML pods ✅
  ML pods:   ONLY land on GPU nodes ✅
  Clean separation, no accidents
```

---

## 🔷 nodeName & nodeSelector (Simpler Alternatives)

```yaml
# nodeName — schedule on EXACT node (rarely used)
spec:
  nodeName: worker-node-1    # hardcoded — not flexible

# nodeSelector — simple label matching (older approach)
spec:
  nodeSelector:
    gpu: "true"              # simple key=value match
                             # no operators, no soft rules
                             # use nodeAffinity for complex rules
```

---

## ✅ Chapter 11 Summary

| Concept | Key Point |
|---|---|
| Taint | Applied to node — repels pods without toleration |
| Toleration | Applied to pod — allows scheduling on tainted node |
| NoSchedule | Don't schedule new pods — existing pods safe |
| NoExecute | Don't schedule + evict existing pods |
| PreferNoSchedule | Soft version of NoSchedule |
| Node Affinity | Pod attracted to nodes with specific labels |
| required | Hard rule — pod stays Pending if no match |
| preferred | Soft rule — scheduler tries but not guaranteed |
| Pod Affinity | Schedule pod near other specific pods |
| Pod Anti-Affinity | Spread pods across nodes — for HA |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is the difference between Taints and Node Affinity?"**
   → Taints are applied to nodes to repel pods — push mechanism. Node Affinity is applied to pods to attract them to specific nodes — pull mechanism. Use both together for dedicated node pools.

2. **"What is the difference between NoSchedule and NoExecute?"**
   → NoSchedule prevents new pods from scheduling on the node but leaves existing pods running. NoExecute also evicts existing pods that don't tolerate the taint.

3. **"How do you ensure a Deployment's replicas spread across nodes?"**
   → Pod Anti-Affinity with `requiredDuringScheduling` and `topologyKey: kubernetes.io/hostname`. Forces each replica onto a different node for high availability.

4. **"What happens if no node matches a required Node Affinity rule?"**
   → Pod stays in Pending state indefinitely until a matching node becomes available. This is why preferred affinity is safer when you can't guarantee node availability.

5. **"How would you set up dedicated GPU nodes in k8s?"**
   → Label GPU nodes, taint them with `dedicated=gpu:NoSchedule`, then add matching toleration and required node affinity to ML pods. Taint keeps non-ML pods off, affinity ensures ML pods land there.

---

Ready for **Chapter 12 → HPA & VPA** — auto-scaling your apps based on real traffic and resource usage? One of the most powerful k8s features and a top interview topic. Say **"next"** 🚀
