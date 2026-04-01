# 📘 Chapter 2 — Kubernetes Architecture
 
> This is the **most important chapter**. Every interview starts here. Every troubleshooting scenario traces back to this. Understand this deeply.

---

## 🔷 The Big Picture

When you set up Kubernetes, you get a **cluster**. A cluster is made of **nodes** (servers/VMs).

Node (in Kubernetes) = A node is simply a machine that runs containers.
That machine can be:
- a virtual machine (VM)
- a physical server
- a cloud instance

```
Kubernetes Cluster
      │
 ┌────┴────┐
 Node1   Node2   Node3
   │        │        │
 Pods     Pods     Pods
```

- Cluster = group of machines
- Node = one machine inside the cluster
- Pod = containers running on that machine
  

These nodes are divided into **2 types:**

```
┌─────────────────────────────────────────────────────────┐
│                    K8s CLUSTER                          │
│                                                         │
│  ┌─────────────────┐      ┌──────────────────────────┐  │
│  │   CONTROL PLANE │      │      WORKER NODES        │  │
│  │  (Master Node)  │      │                          │  │
│  │                 │      │  ┌─────────┐ ┌─────────┐ │  │
│  │  The "Brain"    │─────▶│  │ Node 1  │ │ Node 2  │ │  │
│  │  Manages everything   │  │(runs    │ │(runs    │ │  │
│  │                 │      │  │ pods)   │ │ pods)   │ │  │
│  └─────────────────┘      │  └─────────┘ └─────────┘ │  │
│                           └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

- **Control Plane** = The brain. Makes all decisions.
- **Worker Nodes** = The muscle. Actually runs your containers.

---

## 🔷 Control Plane Components

Let's go one by one. These run on the **master node**.

---

### 1️⃣ API Server (`kube-apiserver`)

```
         You (kubectl)
              │
              ▼
     ┌─────────────────┐
     │   API Server    │  ◀── The FRONT DOOR of Kubernetes
     │  kube-apiserver │
     └─────────────────┘
              │
      talks to everything
```

- **Every single operation** in k8s goes through the API server
- It's a **REST API** under the hood
- When you run `kubectl get pods` → it's an HTTP call to the API server
- It **authenticates**, **validates**, and **processes** every request
- Think of it as the **receptionist** of the entire cluster

**Interview answer:** *"API Server is the central communication hub of k8s. All components talk to each other through it."*

---

### 2️⃣ etcd

```
     ┌─────────────────┐
     │      etcd       │  ◀── The MEMORY of Kubernetes
     │  (key-value DB) │
     └─────────────────┘
```

- A **distributed key-value store** (like a database)
- Stores the **entire cluster state** — every pod, node, config, secret
- If etcd dies → **cluster loses its mind**
- It's the **source of truth**

**Real example:**
```
key: /registry/pods/default/my-app-pod
value: {name: my-app-pod, image: nginx, status: Running, node: worker-1}
```

> ⚠️ In production, etcd is **backed up religiously**. Losing etcd = losing the entire cluster state.

---

### 3️⃣ Scheduler (`kube-scheduler`)

```
     ┌─────────────────┐
     │    Scheduler    │  ◀── The ALLOCATOR
     │ kube-scheduler  │
     └─────────────────┘
```

- Watches for **newly created pods** that have no node assigned
- Decides **which worker node** the pod should run on
- Makes decisions based on:
  - Available CPU & memory on nodes
  - Node selectors / affinity rules
  - Taints and tolerations
  - Resource requests and limits

**Real analogy:**
> You have 3 hotels (nodes). A guest (pod) needs a room with AC + WiFi.
> The scheduler finds the best matching hotel and books the room.

---

### 4️⃣ Controller Manager (`kube-controller-manager`)

```
     ┌──────────────────────┐
     │  Controller Manager  │  ◀── The WATCHDOG
     │                      │
     │  • Node Controller   │
     │  • ReplicaSet Ctrl   │
     │  • Job Controller    │
     │  • Endpoint Ctrl     │
     └──────────────────────┘
```

- Runs a bunch of **controllers** in a single process
- Each controller watches the cluster state and **tries to match desired state**
- Example: You say "I want 3 replicas" → Controller Manager ensures there are always 3

**The control loop (super important concept):**
```
LOOP forever:
  current_state = what's actually running
  desired_state = what you declared in YAML

  if current_state != desired_state:
      take action to fix it
```

This is called the **Reconciliation Loop** — the heart of how k8s works.

---

### 5️⃣ Cloud Controller Manager (CCM)

```
     ┌──────────────────────┐
     │  Cloud Controller    │  ◀── AWS/GCP/Azure bridge
     │     Manager          │
     └──────────────────────┘
```

- Talks to the **cloud provider's API** (AWS, GCP, Azure)
- Handles things like:
  - Creating Load Balancers when you create a `Service` of type `LoadBalancer`
  - Attaching EBS volumes when you create a `PersistentVolume`
  - Managing node lifecycle on cloud VMs
- Only present when running on a cloud provider (not in local clusters)

---

## 🔷 Worker Node Components

These run on **every worker node**.

---

### 1️⃣ Kubelet

```
     ┌─────────────────┐
     │     Kubelet     │  ◀── The NODE AGENT
     └─────────────────┘
```

- Runs on **every worker node**
- It's an **agent** that talks to the API server
- Responsible for:
  - Starting/stopping containers on that node
  - Reporting node health back to master
  - Making sure pods described by API server are actually running
- Talks to the **container runtime** (Docker/containerd) to actually run containers

**Think of it as:** The local manager of each node.

---

### 2️⃣ Kube-proxy

```
     ┌─────────────────┐
     │   Kube-proxy    │  ◀── The NETWORK MANAGER
     └─────────────────┘
```

- Runs on **every worker node**
- Manages **network rules** on the node
- Enables **communication between pods** and between pods and services
- Uses `iptables` or `IPVS` under the hood
- When you hit a Service → kube-proxy routes it to the right pod

---

### 3️⃣ Container Runtime

```
     ┌─────────────────┐
     │Container Runtime│  ◀── Actually RUNS containers
     │ (containerd /   │
     │  CRI-O / Docker)│
     └─────────────────┘
```

- The actual software that runs containers
- k8s doesn't run containers directly — it delegates to the runtime
- Most clusters today use **containerd** (Docker uses it under the hood too)
- Communicates with kubelet via **CRI (Container Runtime Interface)**

> Note: Docker was deprecated as a direct runtime in k8s 1.24+, but **Docker-built images still work** because they follow OCI standards.

---

## 🔷 Full Architecture — Everything Together

```
┌────────────────────────────────────────────────────────────────┐
│                        K8s CLUSTER                             │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  CONTROL PLANE (Master)                  │  │
│  │                                                          │  │
│  │  ┌────────────┐  ┌───────┐  ┌──────────┐  ┌─────────┐    │  │
│  │  │ API Server │  │ etcd  │  │Scheduler │  │Ctrl Mgr │    │  │
│  │  └─────┬──────┘  └───────┘  └──────────┘  └─────────┘    │  │
│  │        │                                                 │  │
│  └────────┼────────────────────────────────────────────── ──┘  │
│           │ (all communication goes via API server)            │
│    ┌──────┴──────────────────────────────────┐                 │
│    │                                         │                 │
│  ┌─▼──────────────────┐   ┌──────────────────▼─ ─┐             │
│  │   WORKER NODE 1    │   │   WORKER NODE 2      │             │
│  │                    │   │                      │             │
│  │ ┌────────────────┐ │   │ ┌────────────────┐   │             │
│  │ │    Kubelet     │ │   │ │    Kubelet     │   │             │
│  │ ├────────────────┤ │   │ ├────────────────┤   │             │
│  │ │   Kube-proxy   │ │   │ │   Kube-proxy   │   │             │
│  │ ├────────────────┤ │   │ ├────────────────┤   │             │
│  │ │Container Runtime│ │   │ │Container Runtime│ │             │
│  │ ├────────────────┤ │   │ ├────────────────┤   │             │
│  │ │  Pod  │  Pod   │ │   │ │  Pod  │  Pod   │   │             │
│  │ └────────────────┘ │   │ └────────────────┘   │             │
│  └────────────────────┘   └──────────────────────┘             │
└────────────────────────────────────────────────────────────────┘
```

---

## 🔷 kubectl — Your Command Line Tool

```
You  ──▶  kubectl  ──▶  API Server  ──▶  Cluster
```

- `kubectl` is the **CLI** to interact with your cluster
- It reads your `~/.kube/config` file to know **which cluster** to talk to
- Every `kubectl` command = an API call to the API Server

**Most used commands (you'll use these daily):**

```bash
# Get resources
kubectl get pods
kubectl get nodes
kubectl get deployments
kubectl get all

# Describe (detailed info)
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Apply a config
kubectl apply -f deployment.yaml

# Delete
kubectl delete pod <pod-name>
kubectl delete -f deployment.yaml

# Logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>   # follow/stream logs

# Exec into a pod (like docker exec)
kubectl exec -it <pod-name> -- /bin/bash

# Check cluster info
kubectl cluster-info
kubectl get nodes
```

---

## 🔷 What Happens When You Deploy an App?

Let's trace the full flow — this is a **killer interview answer:**

```
1. You run: kubectl apply -f app.yaml

2. kubectl sends the YAML to API Server

3. API Server validates it & stores it in etcd
   (etcd now knows: "someone wants 3 pods of nginx")

4. Scheduler sees unscheduled pods
   → picks best worker nodes for each pod
   → updates etcd with node assignments

5. Kubelet on each worker node sees
   "I have a pod assigned to me"
   → tells container runtime to pull image & start container

6. Container starts running ✅

7. Controller Manager watches continuously:
   if a pod dies → creates a new one to match desired state
```

---

## ✅ Chapter 2 Summary

| Component | Role | Where it runs |
|---|---|---|
| API Server | Front door, all communication | Master |
| etcd | Source of truth, cluster state | Master |
| Scheduler | Assigns pods to nodes | Master |
| Controller Manager | Maintains desired state | Master |
| Cloud Controller Manager | Cloud provider integration | Master |
| Kubelet | Node agent, runs pods | Every worker |
| Kube-proxy | Network routing | Every worker |
| Container Runtime | Actually runs containers | Every worker |
| kubectl | CLI to talk to cluster | Your machine |

---

## 🧠 3 Interview Questions You Can Now Answer

1. **"Walk me through k8s architecture"**
   → Control plane (API server, etcd, scheduler, controller manager) + Worker nodes (kubelet, kube-proxy, container runtime)

2. **"What happens when a pod crashes?"**
   → Controller manager detects mismatch between desired & current state → creates a new pod → scheduler assigns it → kubelet starts it

3. **"What is etcd and why is it critical?"**
   → Distributed key-value store, holds entire cluster state, single source of truth, must be backed up

