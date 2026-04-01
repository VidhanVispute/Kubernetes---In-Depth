# 📘 Chapter 3 — Cluster Setup (KIND, Minikube, Kubeadm)

> Theory is done. Time to get your hands dirty. We'll set up **3 types of clusters** — the same ones used in the roadmap.

---

## 🔷 First — The 3 Ways to Run K8s Locally

```
┌─────────────────┬──────────────────────────────┬─────────────────┐ 
│   Tool          │  What it is                  │  Best for       │
├─────────────────┼──────────────────────────────┼─────────────────┤
│ KIND            │ K8s inside Docker containers │ CI/CD, testing  │
│ Minikube        │ K8s in a VM or container     │ Local dev       │
│ Kubeadm         │ Real cluster on real VMs     │ Production-like │
└─────────────────┴──────────────────────────────┴─────────────────┘
```

We'll cover all 3. Let's go.

---

## 🔷 PART 1 — KIND Cluster

### What is KIND?

**KIND = Kubernetes IN Docker**

```
Your Machine
└── Docker
    ├── Container → acts as Master Node
    ├── Container → acts as Worker Node 1
    └── Container → acts as Worker Node 2
```

- Runs entire k8s cluster **inside Docker containers**
- Super lightweight, fast to spin up
- Great for **testing, CI pipelines, local dev**
- No VM needed

---

### 📦 Installation — KIND

**Prerequisites:** Docker must be running ✅ (you already know Docker, so this is easy)

**Step 1 — Install kubectl**
```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
kubectl version --client
```

**Step 2 — Install KIND**
```bash
# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
```

---

### 🚀 Create Your First KIND Cluster

**Simple single-node cluster:**
```bash
# Create cluster
kind create cluster --name my-first-cluster

# Output you'll see:
# Creating cluster "my-first-cluster" ...
# ✓ Ensuring node image (kindest/node:v1.29.2)
# ✓ Preparing nodes
# ✓ Writing configuration
# ✓ Starting control-plane
# ✓ Installing CNI
# ✓ Installing StorageClass
# Set kubectl context to "kind-my-first-cluster"

# Verify
kubectl get nodes
# NAME                             STATUS   ROLES
# my-first-cluster-control-plane  Ready    control-plane
```

---

### 🔧 Multi-Node KIND Cluster (More Realistic)

Create a file `kind-config.yaml`:
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane      # 1 master node
  - role: worker             # worker node 1
  - role: worker             # worker node 2
```

```bash
# Create multi-node cluster
kind create cluster --name multi-node --config kind-config.yaml

# Verify — you should see 3 nodes
kubectl get nodes

# NAME                        STATUS   ROLES
# multi-node-control-plane    Ready    control-plane
# multi-node-worker           Ready    <none>
# multi-node-worker2          Ready    <none>
```

---

### 🛠️ Useful KIND Commands

```bash
# List all clusters
kind get clusters

# Delete a cluster
kind delete cluster --name my-first-cluster

# Load a local Docker image into KIND
# (important! KIND can't pull from local docker by default)
kind load docker-image my-app:latest --name my-cluster

# Get kubeconfig for a cluster
kind get kubeconfig --name my-cluster
```

---

## 🔷 PART 2 — Minikube Cluster

### What is Minikube?

```
Your Machine
└── Minikube
    └── Single VM or Container
        └── Full k8s cluster (master + worker combined)
```

- Runs a **single-node** k8s cluster
- Can use Docker, VirtualBox, or HyperKit as a driver
- Has a great **dashboard UI** built in
- Best for **local development and learning**

---

### 📦 Installation — Minikube

```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify
minikube version
```

---

### 🚀 Start Minikube Cluster

```bash
# Start with Docker driver (since you have Docker)
minikube start --driver=docker

# Output:
# 😄  minikube v1.32.0
# ✨  Using the docker driver
# 📌  Using Docker driver with root privileges
# 🔥  Creating docker container (CPUs=2, Memory=2200MB)
# 🐳  Preparing Kubernetes v1.28.3 on Docker 24.0.7
# ✅  Done! kubectl is now configured to use "minikube"

# Check nodes
kubectl get nodes
# NAME       STATUS   ROLES           AGE
# minikube   Ready    control-plane   60s
```

---

### 🛠️ Useful Minikube Commands

```bash
# Start cluster
minikube start

# Stop cluster (saves state)
minikube stop

# Delete cluster
minikube delete

# Open the Dashboard UI in browser 🔥
minikube dashboard

# SSH into the minikube node
minikube ssh

# Get IP of minikube
minikube ip

# Check status
minikube status

# Start with more resources
minikube start --cpus=4 --memory=8192

# Enable addons (very useful)
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server
```

---

## 🔷 PART 3 — Kubeadm (Production-Like on AWS)

### What is Kubeadm?

- The **official k8s tool** to bootstrap a real cluster on real VMs
- This is closest to **how production clusters are built**
- You manage everything yourself — no magic

```
AWS EC2 VM 1 (t2.medium)  →  Master Node
AWS EC2 VM 2 (t2.micro)   →  Worker Node 1
AWS EC2 VM 3 (t2.micro)   →  Worker Node 2
```

---

### 🖥️ AWS Setup

**Step 1 — Launch EC2 instances**

| Instance | Type | Role |
|---|---|---|
| k8s-master | t2.medium (2 CPU, 4GB RAM minimum) | Control Plane |
| k8s-worker-1 | t2.micro | Worker |
| k8s-worker-2 | t2.micro | Worker |

**Security Group rules — open these ports:**
```
Master Node:
  6443   → Kubernetes API Server
  2379-2380 → etcd
  10250  → Kubelet API
  10257  → kube-scheduler
  10259  → kube-controller-manager

Worker Nodes:
  10250  → Kubelet API
  30000-32767 → NodePort Services
```

---

### ⚙️ Installation on ALL Nodes (Master + Workers)

SSH into each node and run this:

```bash
# Step 1: Disable swap (k8s requires this)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Step 2: Enable kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Step 3: sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Step 4: Install containerd (container runtime)
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Step 5: Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### 🎯 Initialize Master Node ONLY

```bash
# Run ONLY on master node
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# After it completes, set up kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel (network plugin — pods need this to communicate)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Verify master is ready
kubectl get nodes
# NAME         STATUS   ROLES           
# k8s-master   Ready    control-plane   ✅
```

---

### 🔗 Join Worker Nodes

After `kubeadm init`, you'll see a join command. Run it on **each worker node:**

```bash
# This is generated by kubeadm init — copy yours exactly
sudo kubeadm join <MASTER_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If you lost the token, regenerate it:
```bash
# On master node
kubeadm token create --print-join-command
```

**Verify from master:**
```bash
kubectl get nodes
# NAME           STATUS   ROLES
# k8s-master     Ready    control-plane
# k8s-worker-1   Ready    <none>
# k8s-worker-2   Ready    <none>    ✅  All 3 nodes ready!
```

---

## 🔷 Switching Between Clusters

Since you'll have multiple clusters, here's how to manage them:

```bash
# See all contexts (clusters)
kubectl config get-contexts

# Switch to a different cluster
kubectl config use-context kind-my-cluster
kubectl config use-context minikube

# See current context
kubectl config current-context
```

---

## ✅ Chapter 3 Summary

| Tool | How it works | Use case |
|---|---|---|
| KIND | K8s in Docker containers | CI/CD, quick testing |
| Minikube | Single node, local VM/container | Learning, local dev |
| Kubeadm | Real VMs, you manage everything | Production-like, AWS |

**Golden rule for your DevOps job:**
- Interviews → talk about **Kubeadm** (shows depth)
- Day to day testing → use **KIND or Minikube**
- Production → **EKS / GKE / AKS** (managed, Chapter later)

---

## 🧠 Interview Questions You Can Now Answer

1. **"What's the difference between KIND and Minikube?"**
   → KIND runs nodes as Docker containers, better for CI. Minikube runs a full VM, better for local dev.

2. **"How do you bootstrap a production k8s cluster?"**
   → Kubeadm — disable swap, install containerd, kubeadm init on master, join workers, apply CNI plugin.

3. **"What is a CNI plugin and why do you need it?"**
   → Container Network Interface — enables pod-to-pod communication across nodes. Without it, nodes are Ready but pods can't talk to each other.

