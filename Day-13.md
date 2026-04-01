# 📘 Chapter 13 — RBAC (Role Based Access Control)

> Security is not optional in production. RBAC is how you ensure developers can't accidentally delete production databases, CI/CD pipelines only have the permissions they need, and every service only accesses what it must. **Heavily tested in interviews.**

---

## 🔷 The Problem First

```
Without RBAC:

  Junior dev has full cluster access
       ↓
  Accidentally runs: kubectl delete namespace production
       ↓
  Entire production environment gone 💀

  CI/CD pipeline has admin access
       ↓
  Pipeline gets compromised
       ↓
  Attacker has full cluster control 💀

  Every microservice can read every Secret
       ↓
  One compromised service = all secrets exposed 💀

With RBAC:
  Junior dev → read-only in dev namespace only ✅
  CI/CD      → deploy access in specific namespace only ✅
  Service    → only reads its own secrets ✅
```

---

## 🔷 RBAC Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  WHO can do WHAT on WHICH resources                         │
│                                                             │
│  WHO     = Subject  (User, Group, ServiceAccount)          │
│  WHAT    = Verbs    (get, list, create, delete...)         │
│  WHICH   = Resources (pods, secrets, deployments...)       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔷 4 RBAC Objects

```
┌──────────────────┬────────────────────────────────────────────┐
│  Role            │  Permissions within a NAMESPACE            │
├──────────────────┼────────────────────────────────────────────┤
│  ClusterRole     │  Permissions CLUSTER-WIDE                  │
│                  │  (or reusable across namespaces)           │
├──────────────────┼────────────────────────────────────────────┤
│  RoleBinding     │  Binds Role to Subject in a NAMESPACE      │
├──────────────────┼────────────────────────────────────────────┤
│  ClusterRoleBinding │ Binds ClusterRole to Subject            │
│                  │  CLUSTER-WIDE                              │
└──────────────────┴────────────────────────────────────────────┘
```

```
Simple mental model:

  Role/ClusterRole     = JOB DESCRIPTION
                         "This job can read pods and view logs"

  RoleBinding          = EMPLOYMENT CONTRACT
                         "Alice has this job in the dev namespace"

  ClusterRoleBinding   = COMPANY-WIDE CONTRACT
                         "Bob has this job everywhere in the cluster"
```

---

## 🔷 RBAC Subjects — WHO

```
Three types of subjects:

  User          → human identity
                  (k8s has no built-in user DB — uses external auth)
                  e.g. alice, bob, dev-team

  Group         → collection of users
                  e.g. developers, ops-team, qa-team

  ServiceAccount → identity for PODS/applications
                   (pods use this to talk to k8s API)
                   e.g. my-app-sa, ci-cd-sa
```

---

## 🔷 RBAC Verbs — WHAT

```
get       → read a specific resource  (kubectl get pod my-pod)
list      → list resources            (kubectl get pods)
watch     → stream resource changes   (kubectl get pods -w)
create    → create new resource       (kubectl apply)
update    → modify existing resource  (kubectl edit)
patch     → partial update            (kubectl patch)
delete    → remove resource           (kubectl delete)
exec      → execute in container      (kubectl exec)
logs      → view pod logs             (kubectl logs)

*         → ALL verbs (admin access)
```

---

## 🔷 PART 1 — Role & RoleBinding (Namespace-Scoped)

### Creating a Role

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev                    # only applies in 'dev' namespace
rules:
  - apiGroups: [""]                 # "" = core API group (pods, services etc)
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["pods/log"]         # pod logs
    verbs: ["get", "list"]

  - apiGroups: ["apps"]             # apps API group (deployments etc)
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update"]

  - apiGroups: [""]
    resources: ["secrets"]
    verbs: []                       # empty = NO access to secrets ✅
```

---

### 🔷 API Groups — Important to Understand

```
Core resources → apiGroups: [""]
  pods, services, configmaps, secrets,
  namespaces, nodes, persistentvolumes

apps group → apiGroups: ["apps"]
  deployments, replicasets, statefulsets, daemonsets

batch group → apiGroups: ["batch"]
  jobs, cronjobs

networking group → apiGroups: ["networking.k8s.io"]
  ingresses, networkpolicies

autoscaling → apiGroups: ["autoscaling"]
  horizontalpodautoscalers

rbac group → apiGroups: ["rbac.authorization.k8s.io"]
  roles, rolebindings, clusterroles

storage → apiGroups: ["storage.k8s.io"]
  storageclasses
```

---

### Creating a RoleBinding

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: dev                    # binding applies in 'dev' namespace
subjects:
  - kind: User                      # subject type
    name: alice                     # username (from auth system)
    apiGroup: rbac.authorization.k8s.io

  - kind: User
    name: bob                       # multiple users in same binding
    apiGroup: rbac.authorization.k8s.io

roleRef:                            # which role to bind
  kind: Role
  name: developer-role              # must exist in same namespace
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

# Verify
kubectl get roles -n dev
kubectl get rolebindings -n dev

# Check what alice can do
kubectl auth can-i get pods \
  --namespace=dev --as=alice
# yes ✅

kubectl auth can-i delete secrets \
  --namespace=dev --as=alice
# no ✅

kubectl auth can-i get pods \
  --namespace=production --as=alice
# no ✅ (binding only in dev namespace)
```

---

### 🔷 Group Binding — Bind to Entire Team

```yaml
# group-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: dev
subjects:
  - kind: Group
    name: developers                # all users in 'developers' group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

```
New developer joins team?
  Add them to 'developers' group in your auth system
  They automatically get all permissions ✅
  No k8s YAML changes needed
```

---

## 🔷 PART 2 — ClusterRole & ClusterRoleBinding

### When to Use ClusterRole

```
Use ClusterRole when:

  1. Access to cluster-wide resources
     (nodes, persistentvolumes, namespaces)
     These don't belong to any namespace

  2. Same role reused across multiple namespaces
     (don't repeat Role definition in every namespace)

  3. Admin access across entire cluster
```

```yaml
# clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader              # no namespace field!
rules:
  - apiGroups: [""]
    resources: ["nodes"]            # nodes are cluster-scoped
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list"]

  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["pods"]             # across ALL namespaces
    verbs: ["get", "list", "watch"]
```

---

### ClusterRoleBinding — Cluster-Wide Access

```yaml
# clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-reader-binding      # no namespace field
subjects:
  - kind: User
    name: bob-ops                   # ops engineer gets cluster-wide read
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-reader
  apiGroup: rbac.authorization.k8s.io
```

---

### 🔷 ClusterRole + RoleBinding (Reuse Pattern)

```
ClusterRole can be bound using RoleBinding too!

Why?
  Define role ONCE as ClusterRole
  Bind it in specific namespaces via RoleBinding
  No need to repeat Role definition per namespace

┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ClusterRole: developer-role                             │
│  (defined once)                                          │
│         │                                                │
│         ├── RoleBinding in 'dev' namespace → alice       │
│         ├── RoleBinding in 'staging' ns   → alice        │
│         └── RoleBinding in 'prod' ns      → alice        │
│                                                          │
│  vs. creating Role 3 times — much cleaner ✅             │
└──────────────────────────────────────────────────────────┘
```

```yaml
# rolebinding-using-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-dev-binding
  namespace: dev                    # scoped to dev namespace
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole                 # referencing ClusterRole
  name: developer-role              # but scoped to namespace via RoleBinding
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔷 PART 3 — ServiceAccounts

### What is a ServiceAccount?

```
Users    = humans authenticating to cluster
ServiceAccounts = PODS authenticating to cluster

Every pod that calls the k8s API needs an identity
→ That identity is a ServiceAccount

Examples:
  CI/CD pipeline pod → needs ServiceAccount to deploy apps
  Prometheus pod     → needs ServiceAccount to scrape metrics
  Your app pod       → needs ServiceAccount to read ConfigMaps
```

---

### 🔷 Default ServiceAccount

```bash
# Every namespace has a default ServiceAccount
kubectl get serviceaccounts -n dev
# NAME      SECRETS   AGE
# default   0         5d

# Every pod gets the default SA automatically
# Default SA has minimal permissions (read-only in most setups)
kubectl describe pod my-pod -n dev | grep "Service Account"
# Service Account:  default
```

---

### 🛠️ Create Custom ServiceAccount

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: dev
```

```bash
kubectl apply -f serviceaccount.yaml
```

---

### 🛠️ Bind ServiceAccount to Role

```yaml
# sa-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: dev
subjects:
  - kind: ServiceAccount            # subject is ServiceAccount
    name: app-service-account
    namespace: dev                  # SA namespace required
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

---

### 🛠️ Use ServiceAccount in a Pod

```yaml
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: dev
spec:
  serviceAccountName: app-service-account   # assign SA to pod
  containers:
    - name: app
      image: my-app:latest
```

```
The pod now:
  → Gets a token mounted at /var/run/secrets/kubernetes.io/serviceaccount/
  → Can use that token to call the k8s API
  → Has exactly the permissions defined in the bound Role
```

---

## 🔷 PART 4 — Real World RBAC Patterns

### Pattern 1 — Developer Access (Most Common)

```yaml
# dev-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
  # Can view everything
  - apiGroups: ["", "apps", "batch"]
    resources:
      - pods
      - deployments
      - services
      - configmaps
      - jobs
      - cronjobs
    verbs: ["get", "list", "watch"]

  # Can view logs
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]

  # Can exec into pods (for debugging)
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]

  # Can deploy (create/update deployments)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["create", "update", "patch"]

  # CANNOT touch secrets
  # CANNOT delete anything
  # CANNOT access production namespace
```

---

### Pattern 2 — CI/CD Pipeline ServiceAccount

```yaml
# cicd-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
  # Can manage deployments (rolling updates)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]

  # Can manage configmaps (config updates)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]

  # Can view pods (verify deployment health)
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

  # CANNOT delete deployments
  # CANNOT touch secrets
  # CANNOT access other namespaces
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-sa
    namespace: production
roleRef:
  kind: Role
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

---

### Pattern 3 — Read-Only Cluster Monitoring

```yaml
# monitoring-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - pods
      - services
      - endpoints
      - namespaces
    verbs: ["get", "list", "watch"]

  - apiGroups: ["apps"]
    resources:
      - deployments
      - statefulsets
      - daemonsets
    verbs: ["get", "list", "watch"]

  - nonResourceURLs:
      - "/metrics"                  # Prometheus scrape endpoint
    verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 🔷 PART 5 — RBAC Debugging

### Most Useful Commands

```bash
# Can I do this action?
kubectl auth can-i create deployments -n dev
# yes/no

# Can alice do this?
kubectl auth can-i delete pods \
  -n production --as=alice
# no

# Can the cicd-sa ServiceAccount do this?
kubectl auth can-i update deployments \
  -n production \
  --as=system:serviceaccount:production:cicd-sa
# yes

# What can alice do? (list all permissions)
kubectl auth can-i --list --as=alice -n dev

# Who can delete pods in production?
kubectl who-can delete pods -n production
# (requires kubectl who-can plugin)

# View all roles in a namespace
kubectl get roles -n dev
kubectl get rolebindings -n dev

# View all cluster roles
kubectl get clusterroles
kubectl get clusterrolebindings

# Describe to see full rules
kubectl describe role developer -n dev
kubectl describe rolebinding dev-team-binding -n dev
```

---

### Common RBAC Errors

```bash
# Error you'll see when RBAC blocks something:
# Error from server (Forbidden):
# pods is forbidden:
# User "alice" cannot list resource "pods"
# in API group "" in the namespace "production"

# Debug steps:
# 1. Check what role alice has
kubectl get rolebindings -n production \
  -o yaml | grep alice

# 2. Check what the role allows
kubectl describe role alice-role -n production

# 3. Test specific permission
kubectl auth can-i list pods \
  -n production --as=alice

# 4. Check if using wrong namespace
kubectl auth can-i list pods \
  -n dev --as=alice         # maybe binding is in dev not production
```

---

## 🔷 RBAC Best Practices

```
Principle of Least Privilege:
  ✅ Give MINIMUM permissions needed
  ✅ Prefer Role over ClusterRole where possible
  ✅ Never give * (wildcard) verbs in production
  ✅ Never give admin ClusterRoleBinding to users/SAs

ServiceAccount best practices:
  ✅ Create dedicated SA per application
  ✅ Never use default SA for apps
  ✅ Disable automounting if SA token not needed:
     automountServiceAccountToken: false

Audit regularly:
  ✅ Review who has ClusterAdmin
  ✅ Review CI/CD pipeline permissions
  ✅ Remove permissions when not needed

Dangerous combinations to avoid:
  ❌ create + exec on pods = can run anything
  ❌ patch on deployments = can change image = code injection
  ❌ get secrets cluster-wide = can read all passwords
```

---

## ✅ Chapter 13 Summary

| Concept | Key Point |
|---|---|
| Role | Permissions within one namespace |
| ClusterRole | Permissions cluster-wide or reusable |
| RoleBinding | Attaches Role to Subject in a namespace |
| ClusterRoleBinding | Attaches ClusterRole to Subject cluster-wide |
| User | Human identity — from external auth system |
| Group | Collection of users |
| ServiceAccount | Pod identity — for app-to-k8s-API auth |
| Verbs | get, list, watch, create, update, patch, delete |
| apiGroups | "" = core, "apps", "batch", "networking.k8s.io" |
| Least Privilege | Always minimum permissions needed |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is RBAC in Kubernetes?"**
   → Role Based Access Control. Controls who can do what on which resources. Uses Roles/ClusterRoles to define permissions and RoleBindings/ClusterRoleBindings to assign them to Users, Groups, or ServiceAccounts.

2. **"What is the difference between Role and ClusterRole?"**
   → Role is namespace-scoped — permissions apply only within one namespace. ClusterRole is cluster-wide — for cluster-scoped resources like nodes, or reusable across namespaces via RoleBindings.

3. **"What is a ServiceAccount and why do pods need one?"**
   → ServiceAccount is an identity for pods to authenticate to the k8s API. Pods use it to call k8s API — for example Prometheus needs SA to scrape metrics, CI/CD needs SA to deploy apps.

4. **"How would you give a CI/CD pipeline permission to deploy?"**
   → Create a ServiceAccount, create a Role with update/patch on deployments, bind them with RoleBinding in the target namespace. Mount the SA in the pipeline pod.

5. **"What is the Principle of Least Privilege in k8s?"**
   → Give each user/SA only the minimum permissions they need. Prefer namespace-scoped Roles over ClusterRoles. Never use wildcards in production. Audit and rotate permissions regularly.

---

Ready for **Chapter 14 → Monitoring & Logging** (Kubernetes Dashboard, Prometheus, Grafana) — how to actually see what's happening inside your cluster in real time? Say **"next"** 🚀
