# 📘 Chapter 5 — Deployments, ReplicaSets & Rolling Updates

> This is where k8s gets **powerful**. Deployments are what you'll work with 90% of the time as a DevOps engineer. Master this chapter completely.

---

## 🔷 The Problem With Naked Pods (Recap)

```
You create a Pod manually
        ↓
Pod crashes / node dies
        ↓
Pod is GONE forever 💀
        ↓
No one restarts it
        ↓
Your app is down
```

You need something that:
- **Keeps pods alive** always
- **Scales** pods up/down
- **Updates** pods without downtime
- **Rolls back** if something breaks

That thing is a **Deployment.**

---

## 🔷 The Relationship — Deployment → ReplicaSet → Pod

```
┌─────────────────────────────────────────────────┐
│                  DEPLOYMENT                     │
│  (manages updates, rollbacks, scaling strategy) │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │              REPLICASET                 │   │
│   │  (ensures N copies of pod always run)   │   │
│   │                                         │   │
│   │   ┌───────┐  ┌───────┐  ┌───────┐       │   │
│   │   │  Pod  │  │  Pod  │  │  Pod  │       │   │
│   │   │  v1   │  │  v1   │  │  v1   │       │   │
│   │   └───────┘  └───────┘  └───────┘       │   │
│   └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

- **Deployment** → the top-level controller you create
- **ReplicaSet** → automatically created by Deployment, ensures pod count
- **Pod** → what actually runs your container

> You never create ReplicaSets manually. Deployment creates and manages them for you.

---

## 🔷 PART 1 — Deployments

### Your First Deployment YAML

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 3                    # keep 3 pods running always
  selector:
    matchLabels:
      app: nginx                 # manages pods with this label
  template:                      # pod template — this is the pod spec
    metadata:
      labels:
        app: nginx               # must match selector above
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

```bash
# Apply it
kubectl apply -f deployment.yaml

# Check deployment
kubectl get deployments -n dev

# NAME               READY   UP-TO-DATE   AVAILABLE
# nginx-deployment   3/3     3            3          ✅

# Check pods it created
kubectl get pods -n dev

# NAME                                READY   STATUS
# nginx-deployment-7d9f9c7b5f-abc12   1/1     Running
# nginx-deployment-7d9f9c7b5f-def34   1/1     Running
# nginx-deployment-7d9f9c7b5f-ghi56   1/1     Running
```

---

### 🔷 Self-Healing in Action

```bash
# Delete one pod manually
kubectl delete pod nginx-deployment-7d9f9c7b5f-abc12 -n dev

# Watch what happens
kubectl get pods -n dev -w

# NAME                                READY   STATUS
# nginx-deployment-7d9f9c7b5f-def34   1/1     Running
# nginx-deployment-7d9f9c7b5f-ghi56   1/1     Running
# nginx-deployment-7d9f9c7b5f-abc12   0/1     Terminating
# nginx-deployment-7d9f9c7b5f-xyz99   0/1     Pending      ← NEW pod!
# nginx-deployment-7d9f9c7b5f-xyz99   1/1     Running      ✅
```

> ReplicaSet saw 2 pods instead of 3 → immediately created a replacement.
> **This is self-healing.**

---

### 🛠️ Deployment Commands

```bash
# Create
kubectl apply -f deployment.yaml

# List deployments
kubectl get deployments -n dev
kubectl get deploy -n dev          # shorthand

# Detailed info
kubectl describe deployment nginx-deployment -n dev

# Scale up/down
kubectl scale deployment nginx-deployment --replicas=5 -n dev

# Edit live (opens in vim)
kubectl edit deployment nginx-deployment -n dev

# Delete
kubectl delete deployment nginx-deployment -n dev

# Get the ReplicaSet it created
kubectl get replicasets -n dev
kubectl get rs -n dev              # shorthand
```

---

## 🔷 PART 2 — Rolling Updates

### The Problem Rolling Updates Solve

```
Old way (before k8s):
  Stop app → Deploy new version → Start app
  = DOWNTIME ❌

K8s Rolling Update:
  Replace pods one by one
  = ZERO DOWNTIME ✅
```

### How Rolling Update Works

```
Initial state: 3 pods running v1
─────────────────────────────────

Step 1: Start 1 new pod with v2
  [v1] [v1] [v1] [v2-starting]

Step 2: v2 pod is ready, kill 1 v1 pod
  [v1] [v1] [v2]

Step 3: Start another v2 pod
  [v1] [v1] [v2] [v2-starting]

Step 4: Ready, kill another v1
  [v1] [v2] [v2]

Step 5: Repeat until done
  [v2] [v2] [v2]  ✅ Full rollout, zero downtime
```

---

### 🔷 Rolling Update Strategy in YAML

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate          # default strategy
    rollingUpdate:
      maxSurge: 1                # max extra pods during update
      maxUnavailable: 1          # max pods that can be down during update
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

**What these mean:**
```
replicas: 3
maxSurge: 1        → can have max 4 pods during update (3+1)
maxUnavailable: 1  → at least 2 pods always available

So during update:
  Min pods running: 3 - 1 = 2   (never goes below this)
  Max pods running: 3 + 1 = 4   (never goes above this)
```

---

### 🛠️ Trigger a Rolling Update

```bash
# Method 1 — Update image via command
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.26 -n dev

# Method 2 — Edit the YAML and re-apply
# change image: nginx:1.25 → image: nginx:1.26
kubectl apply -f deployment.yaml

# Watch the rollout happen in real time 🔥
kubectl rollout status deployment/nginx-deployment -n dev

# Output:
# Waiting for deployment "nginx-deployment" rollout to finish:
# 1 out of 3 new replicas have been updated...
# 2 out of 3 new replicas have been updated...
# 3 out of 3 new replicas have been updated...
# deployment "nginx-deployment" successfully rolled out ✅
```

---

### 🔷 Rollback — When Things Go Wrong

```bash
# Check rollout history
kubectl rollout history deployment/nginx-deployment -n dev

# REVISION  CHANGE-CAUSE
# 1         <none>     ← original v1
# 2         <none>     ← update to v2 (currently active)

# Rollback to previous version
kubectl rollout undo deployment/nginx-deployment -n dev

# Rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment \
  --to-revision=1 -n dev

# Verify rollback
kubectl rollout status deployment/nginx-deployment -n dev
```

> **Pro tip:** Add `--record` flag when applying to track change cause in history:
> `kubectl apply -f deployment.yaml --record`

---

### 🔷 Recreate Strategy (Alternative)

```yaml
strategy:
  type: Recreate    # kills ALL old pods first, then creates new ones
                    # causes downtime but ensures clean switch
                    # use when old & new versions can't run simultaneously
```

```
Recreate:
  Kill [v1] [v1] [v1]
  ↓ (downtime here)
  Start [v2] [v2] [v2]

Use case: Database schema migrations that break old app version
```

---

## 🔷 PART 3 — ReplicaSets (Deep Dive)

### What is a ReplicaSet?

```yaml
# replicaset.yaml (you'll rarely write this directly)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

ReplicaSet has **one job:**
> "Make sure exactly N pods with matching labels are always running"

---

### 🔷 Labels & Selectors — The Glue of K8s

This is **critical** to understand deeply.

```
Labels  = tags you put on pods
          (key: value pairs)

Selectors = how controllers FIND their pods
            (give me all pods with label app=nginx)
```

```yaml
# Pod has labels:
metadata:
  labels:
    app: nginx        ← label
    env: production   ← label
    version: v1       ← label

# ReplicaSet/Deployment uses selector to find its pods:
spec:
  selector:
    matchLabels:
      app: nginx      ← find all pods with this label
```

```bash
# Filter pods by label
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,env=production    # multiple labels

# Show labels on pods
kubectl get pods --show-labels
```

**Why labels matter:**

```
┌──────────────────────────────────────────┐
│            K8s CLUSTER                   │
│                                          │
│  Pod [app=nginx, env=prod]   ←──┐        │
│  Pod [app=nginx, env=prod]   ←──┼── Deployment selects these
│  Pod [app=nginx, env=prod]   ←──┘        │
│                                          │
│  Pod [app=redis, env=prod]               │
│  Pod [app=nginx, env=dev]    ← different │
└──────────────────────────────────────────┘
```

> Labels are how **Services find pods**, how **Deployments manage pods**, how **monitoring tools scrape pods**. Labels are everywhere in k8s.

---

## 🔷 PART 4 — DaemonSets

### What is a DaemonSet?

```
ReplicaSet  → run N copies of a pod anywhere in cluster
DaemonSet   → run EXACTLY 1 copy of a pod on EVERY node
```

```
┌─────────────────────────────────────────────┐
│                CLUSTER                      │
│                                             │
│  Node 1 → [log-collector pod]   ← DaemonSet │
│  Node 2 → [log-collector pod]   ← DaemonSet │
│  Node 3 → [log-collector pod]   ← DaemonSet │
│                                             │
│  Add Node 4 → [log-collector pod] auto! ✅  │
└─────────────────────────────────────────────┘
```

**Real use cases:**
- Log collectors (Fluentd, Filebeat) — every node must ship logs
- Monitoring agents (Prometheus Node Exporter) — every node must be monitored
- Network plugins (Flannel, Calico) — every node needs networking
- Security agents — every node must be scanned

```yaml
# daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: fluentd
          image: fluentd:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log       # access node logs
      volumes:
        - name: varlog
          hostPath:
            path: /var/log             # mount node's log dir
```

```bash
# Apply and verify
kubectl apply -f daemonset.yaml
kubectl get daemonsets -n monitoring

# NAME            DESIRED   CURRENT   READY   NODE SELECTOR
# log-collector   3         3         3       <none>        ✅
# (one per node automatically)
```

---

## 🔷 ReplicaSet vs DaemonSet vs Deployment

```
┌──────────────┬────────────────────────────┬──────────────────────┐
│   Resource   │   What it does             │   Use case           │
├──────────────┼────────────────────────────┼──────────────────────┤
│ ReplicaSet   │ Maintains N pod replicas   │ Rarely used directly │
│              │ anywhere in cluster        │ (Deployment does it) │
├──────────────┼────────────────────────────┼──────────────────────┤
│ Deployment   │ Manages ReplicaSets        │ Stateless apps       │
│              │ + rolling updates          │ Web servers, APIs    │
│              │ + rollbacks                │                      │
├──────────────┼────────────────────────────┼──────────────────────┤
│ DaemonSet    │ 1 pod per node             │ Log collectors       │
│              │ auto-adjusts with nodes    │ Monitoring agents    │
│              │                            │ Network plugins      │
└──────────────┴────────────────────────────┴──────────────────────┘
```

---

## ✅ Chapter 5 Summary

| Concept | Key Point |
|---|---|
| Deployment | Top-level controller — manages ReplicaSets + updates |
| ReplicaSet | Ensures N pods always running — auto-created by Deployment |
| Self-healing | Pod dies → ReplicaSet immediately creates replacement |
| Rolling Update | Replace pods one by one — zero downtime |
| Recreate | Kill all old pods then create new — causes downtime |
| Rollback | `kubectl rollout undo` — instant recovery |
| Labels | Key-value tags on resources |
| Selectors | How controllers find and manage their pods |
| DaemonSet | 1 pod per node — for infra-level workloads |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What's the difference between a Deployment and a ReplicaSet?"**
   → ReplicaSet ensures N pod replicas exist. Deployment manages ReplicaSets and adds rolling update + rollback capability. You always use Deployment in practice.

2. **"How does a rolling update work in k8s?"**
   → Gradually replaces old pods with new ones using maxSurge and maxUnavailable settings. Ensures minimum availability throughout. Zero downtime.

3. **"How do you rollback a bad deployment?"**
   → `kubectl rollout undo deployment/<name>` — reverts to previous ReplicaSet instantly.

4. **"What is a DaemonSet and when do you use it?"**
   → Ensures exactly one pod runs on every node. Used for log collectors, monitoring agents, network plugins — anything that must run cluster-wide per node.

5. **"How do Labels and Selectors work?"**
   → Labels are key-value pairs on resources. Selectors filter resources by labels. Deployments use selectors to know which pods they own. Services use selectors to know which pods to send traffic to.

