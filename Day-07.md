# 📘 Chapter 7 — Storage (PV, PVC, StorageClasses)

> Storage is where most beginners get confused. By the end of this chapter you'll understand it completely — and be able to explain it clearly in interviews.

--- 

## 🔷 The Problem First

```
Containers are ephemeral by nature:

  Container starts  → gets fresh filesystem
  Container crashes → filesystem WIPED
  Container restarts → starts fresh again

This is fine for stateless apps.
This is a disaster for:
  → Databases        (need data to survive restarts)
  → File uploads     (need files to persist)
  → ML model files   (need weights to persist)
  → Log files        (need logs to survive pod death)
```

Kubernetes solves this with **Volumes.**

---

## 🔷 The 3-Layer Storage System

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   ADMIN creates  →  PersistentVolume (PV)           │
│                      "Here's 100Gi of storage"      │
│                                                     │
│   DEV claims     →  PersistentVolumeClaim (PVC)     │
│                      "I need 10Gi of storage"       │
│                                                     │
│   POD uses       →  mounts PVC into container       │
│                      "/data directory in container" │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Real world analogy:**
```
PV  = Physical hard drive in a data center
      (Admin buys and racks the drive)

PVC = Request for a portion of that drive
      (Dev says "I need 10GB")

Pod = Application that uses the drive
      (Mounts it at /data inside container)
```

---

## 🔷 PART 1 — Persistent Volumes (PV)

### What is a PV?

A PersistentVolume is a **piece of storage** in the cluster that has been:
- Provisioned by an admin **manually** (static provisioning)
- OR provisioned **automatically** by StorageClass (dynamic provisioning)

It exists **independently of any pod** — even if the pod dies, the PV and its data survive.

---

### 🛠️ PV YAML — Static Provisioning

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi                  # size of this volume
  accessModes:
    - ReadWriteOnce                # access mode (see below)
  persistentVolumeReclaimPolicy: Retain   # what to do when PVC released
  storageClassName: manual         # ties to a StorageClass
  hostPath:                        # storage type (local node path)
    path: "/mnt/data"              # path on the NODE (not container)
```

---

### 🔷 Access Modes — Critical to Understand

```
ReadWriteOnce (RWO)
  → mounted as read-write by ONE node at a time
  → most common — used for databases
  → Example: MySQL, PostgreSQL

ReadOnlyMany (ROX)
  → mounted as read-only by MANY nodes simultaneously
  → Example: static config files, ML models

ReadWriteMany (RWX)
  → mounted as read-write by MANY nodes simultaneously
  → Example: shared file storage (NFS, EFS)
  → NOT supported by all storage backends (EBS doesn't support it)
```

```
AWS Storage support:
  EBS (Elastic Block Store) → RWO only
  EFS (Elastic File System) → RWX ✅
  
GCP:
  Persistent Disk → RWO
  Filestore       → RWX ✅
```

---

### 🔷 Reclaim Policy — What Happens After PVC is Deleted

```
Retain   → PV kept as-is, data preserved
           Admin must manually clean up
           Use when: data is critical, manual review needed

Delete   → PV and underlying storage automatically deleted
           Use when: dev/test environments, cloud storage
           Default for dynamically provisioned volumes

Recycle  → Basic scrub (rm -rf), then made available again
           DEPRECATED — don't use
```

---

### 🔷 PV Lifecycle

```
Available  → PV created, not claimed by any PVC yet

Bound      → PVC found a matching PV, now linked together

Released   → PVC deleted, but PV not yet reclaimed
             (data still there, can't be claimed by new PVC yet)

Failed     → Automatic reclamation failed
```

---

## 🔷 PART 2 — Persistent Volume Claims (PVC)

### What is a PVC?

A PVC is a **request for storage** by a user/pod.

```
Developer doesn't need to know:
  ❌ What type of disk is it?
  ❌ Where is it physically?
  ❌ Which node is it on?

Developer just says:
  ✅ "I need 5Gi"
  ✅ "I need ReadWriteOnce"
  ✅ "Give me storage"

K8s finds a matching PV and binds them together.
```

---

### 🛠️ PVC YAML

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce          # must match PV's access mode
  storageClassName: manual   # must match PV's storageClassName
  resources:
    requests:
      storage: 5Gi           # requesting 5Gi (PV has 10Gi — that's ok)
```

```bash
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml

# Check binding
kubectl get pv
# NAME    CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM
# my-pv   10Gi      RWO           Retain          Bound   dev/my-pvc ✅

kubectl get pvc -n dev
# NAME    STATUS  VOLUME  CAPACITY  ACCESS MODES
# my-pvc  Bound   my-pv   10Gi      RWO          ✅
```

---

### 🛠️ Using PVC in a Pod

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  namespace: dev
spec:
  containers:
    - name: mysql
      image: mysql:8.0
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: "password"
      volumeMounts:
        - name: mysql-storage      # must match volume name below
          mountPath: /var/lib/mysql  # where to mount inside container
  volumes:
    - name: mysql-storage          # volume name
      persistentVolumeClaim:
        claimName: my-pvc          # reference to the PVC
```

```bash
kubectl apply -f pod-with-pvc.yaml

# Verify mount
kubectl exec -it mysql-pod -n dev -- df -h
# Filesystem      Size  Used Avail
# /dev/sda1       5.0G  1.2G  3.8G  /var/lib/mysql  ✅

# Data test
kubectl exec -it mysql-pod -n dev -- mysql -u root -ppassword \
  -e "CREATE DATABASE testdb;"

# Delete pod
kubectl delete pod mysql-pod -n dev

# Recreate pod
kubectl apply -f pod-with-pvc.yaml

# Data still there!
kubectl exec -it mysql-pod -n dev -- mysql -u root -ppassword \
  -e "SHOW DATABASES;"
# testdb  ← still exists ✅  DATA SURVIVED POD DEATH
```

---

## 🔷 PART 3 — StorageClasses

### The Problem With Static Provisioning

```
Static PV provisioning flow:

  1. Admin manually creates PV
  2. Dev creates PVC
  3. K8s binds them

Problems:
  → Admin must pre-create PVs before devs need them
  → Wasted storage (create 100 PVs, only 30 used)
  → Doesn't scale in large clusters
  → Manual work every time
```

**Solution: StorageClass + Dynamic Provisioning**

---

### 🔷 What is a StorageClass?

```
StorageClass = a template for automatic PV creation

When a PVC is created with a StorageClass:
  1. PVC submitted
  2. StorageClass provisioner kicks in automatically
  3. Actual storage created (EBS volume, GCE disk etc.)
  4. PV created automatically
  5. PVC bound to new PV

Zero manual work from admin ✅
```

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  StorageClass: "fast-ssd"                            │
│    provisioner: ebs.csi.aws.com                      │
│    type: gp3                                         │
│    reclaimPolicy: Delete                             │
│                                                      │
│  PVC requests "fast-ssd" StorageClass                │
│       ↓                                              │
│  AWS EBS volume automatically created               │
│       ↓                                              │
│  PV automatically created & bound to PVC ✅          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

### 🛠️ StorageClass YAML

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com        # AWS EBS CSI driver
parameters:
  type: gp3                          # EBS volume type
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete                # delete EBS when PVC deleted
allowVolumeExpansion: true           # allow resizing PVC later
volumeBindingMode: WaitForFirstConsumer  # wait until pod scheduled
```

---

### 🔷 Common Provisioners

```
AWS:
  ebs.csi.aws.com          → EBS (block storage, RWO)
  efs.csi.aws.com          → EFS (file storage, RWX)

GCP:
  pd.csi.storage.gke.io    → Persistent Disk

Azure:
  disk.csi.azure.com       → Azure Disk
  file.csi.azure.com       → Azure Files

Local/On-prem:
  kubernetes.io/no-provisioner → no dynamic provisioning
  rancher.io/local-path        → local path provisioner
  docker.io/hostpath           → for Docker Desktop
```

---

### 🛠️ PVC With StorageClass (Dynamic Provisioning)

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd      # reference the StorageClass
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f dynamic-pvc.yaml

# PV is automatically created!
kubectl get pvc -n dev
# NAME         STATUS  VOLUME                                    CAPACITY
# dynamic-pvc  Bound   pvc-8f2d1a3b-xxxx-xxxx-xxxx-xxxxxxxxxx   20Gi  ✅

kubectl get pv
# NAME                                      CAPACITY  STATUS
# pvc-8f2d1a3b-xxxx-xxxx-xxxx-xxxxxxxxxx   20Gi      Bound   ✅
# (automatically created by StorageClass!)
```

---

### 🔷 Default StorageClass

```bash
# See all storage classes in cluster
kubectl get storageclass
kubectl get sc       # shorthand

# NAME            PROVISIONER             DEFAULT
# gp2             ebs.csi.aws.com         ✅ (default)
# fast-ssd        ebs.csi.aws.com

# If PVC doesn't specify storageClassName
# → default StorageClass is used automatically
```

---

### 🔷 Resizing a PVC (StorageClass must allow it)

```bash
# Edit PVC to increase size
kubectl edit pvc dynamic-pvc -n dev
# change storage: 20Gi → storage: 50Gi

# Check resize status
kubectl describe pvc dynamic-pvc -n dev
# Conditions:
#   FileSystemResizePending → pod restart needed to complete resize
#
# Restart pod to complete filesystem resize
kubectl delete pod mysql-pod -n dev
```

---

## 🔷 Full Storage Flow — Everything Together

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  STATIC PROVISIONING                                            │
│  Admin: creates PV manually                                     │
│  Dev:   creates PVC  →  k8s binds PVC to matching PV           │
│  Pod:   mounts PVC   →  reads/writes to persistent storage      │
│                                                                 │
│  DYNAMIC PROVISIONING (recommended)                             │
│  Admin: creates StorageClass once                               │
│  Dev:   creates PVC with storageClassName                       │
│         →  provisioner auto-creates PV + actual storage         │
│         →  PVC auto-bound                                       │
│  Pod:   mounts PVC   →  reads/writes to persistent storage      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🔷 Storage on AWS EKS — Real World Setup

```bash
# Step 1: Install EBS CSI driver on EKS
eksctl create addon --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --service-account-role-arn arn:aws:iam::ACCOUNT:role/EBSCSIRole

# Step 2: Create StorageClass
kubectl apply -f storageclass.yaml

# Step 3: Set as default (optional)
kubectl patch storageclass fast-ssd \
  -p '{"metadata":{"annotations":
  {"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Now any PVC without storageClassName
# automatically gets fast-ssd EBS volume ✅
```

---

## ✅ Chapter 7 Summary

| Concept | Key Point |
|---|---|
| PersistentVolume (PV) | Actual storage resource in cluster |
| PersistentVolumeClaim (PVC) | Request for storage by a pod/dev |
| StorageClass | Template for dynamic PV provisioning |
| Static Provisioning | Admin creates PVs manually |
| Dynamic Provisioning | StorageClass auto-creates PV when PVC created |
| RWO | One node reads/writes — databases |
| RWX | Many nodes read/write — shared files |
| Reclaim Policy | Retain = keep data, Delete = auto cleanup |
| volumeClaimTemplates | StatefulSet gives each pod its own PVC |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is the difference between PV and PVC?"**
   → PV is the actual storage resource provisioned by admin or automatically. PVC is the request for storage by the developer. K8s binds a PVC to a matching PV.

2. **"What is a StorageClass and why is it better than static PVs?"**
   → StorageClass enables dynamic provisioning — automatically creates PVs on demand. No manual admin work, no wasted pre-provisioned storage, scales automatically.

3. **"What access modes does EBS support?"**
   → Only ReadWriteOnce — one node at a time. For ReadWriteMany you need EFS.

4. **"What happens to a PVC when a StatefulSet is deleted?"**
   → PVCs are NOT deleted — they persist to protect data. Must be manually deleted.

5. **"What is volumeBindingMode: WaitForFirstConsumer?"**
   → Delays PV creation until a pod actually needs it. Ensures volume is created in same availability zone as the pod — critical on AWS where EBS volumes are AZ-specific.

---

Ready for **Chapter 8 → Services & Networking** — how pods talk to each other and the outside world? This is where `ClusterIP`, `NodePort`, `LoadBalancer` and `Ingress` all come together. Say **"next"** 🚀
