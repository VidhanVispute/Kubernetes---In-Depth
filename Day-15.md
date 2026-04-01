# 📘 Chapter 15 — Helm, CRDs & Operators

> This chapter covers the tools that power **real production deployments**. Helm makes complex deployments repeatable. CRDs extend k8s itself. Operators automate entire application lifecycles. Every senior DevOps/Platform engineer uses all three daily.

---

## 🔷 The Problem First

```
You need to deploy Prometheus to your cluster.

Without Helm:
  → 30+ YAML files
  → ServiceAccounts, ClusterRoles, ClusterRoleBindings
  → ConfigMaps, Secrets, Deployments, Services
  → Custom Resource Definitions
  → All interdependent, order matters
  → Different values for dev vs prod
  → Upgrade = manually diff 30 files
  → Rollback = manually revert 30 files
  → 2-3 hours of work 😰

With Helm:
  helm install monitoring prometheus-community/kube-prometheus-stack
  → Done in 30 seconds ✅
  → Upgrade:  helm upgrade monitoring ...
  → Rollback: helm rollback monitoring
  → Different envs: helm install -f prod-values.yaml
```

---

## 🔷 PART 1 — Helm

### What is Helm?

```
Helm = Package manager for Kubernetes

  Like apt for Ubuntu
  Like npm for Node.js
  Like pip for Python

  But for k8s YAML manifests

Key concepts:
  Chart    → package of k8s manifests (like a npm package)
  Release  → installed instance of a chart
  Repo     → collection of charts (like npm registry)
  Values   → configuration that customizes a chart
```

---

### 🔷 Helm Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Helm Chart (on chart repository)                          │
│  ┌─────────────────────────────────┐                       │
│  │  Chart.yaml    (metadata)       │                       │
│  │  values.yaml   (defaults)       │                       │
│  │  templates/                     │                       │
│  │    deployment.yaml              │                       │
│  │    service.yaml                 │                       │
│  │    ingress.yaml                 │                       │
│  │    configmap.yaml               │                       │
│  │    _helpers.tpl                 │                       │
│  └─────────────────────────────────┘                       │
│            │                                               │
│            │  helm install + your values.yaml              │
│            ▼                                               │
│  Rendered k8s manifests → applied to cluster               │
│  Release tracked in cluster (stored as Secrets)            │
│                                                            │
└─────────────────────────────────────────────────────────────┘
```

---

### 🛠️ Install Helm

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
# version.BuildInfo{Version:"v3.14.0"...}
```

---

### 🛠️ Helm Basic Commands

```bash
# ─── REPOS ──────────────────────────────────────────────
# Add a chart repository
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo add ingress-nginx \
  https://kubernetes.github.io/ingress-nginx

helm repo add grafana \
  https://grafana.github.io/helm-charts

helm repo add bitnami \
  https://charts.bitnami.com/bitnami

# Update all repos (like apt-get update)
helm repo update

# List added repos
helm repo list

# Search for charts
helm search repo prometheus
helm search hub nginx          # search Artifact Hub (public)


# ─── INSTALL ────────────────────────────────────────────
# Basic install
helm install <release-name> <chart>

helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Install with custom values
helm install nginx ingress-nginx/ingress-nginx \
  --namespace ingress \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.resources.requests.cpu=100m


# ─── LIST & STATUS ──────────────────────────────────────
# List all releases
helm list
helm list -n monitoring
helm list --all-namespaces

# Release status
helm status monitoring -n monitoring

# What k8s resources a release created
helm get manifest monitoring -n monitoring


# ─── UPGRADE ────────────────────────────────────────────
# Upgrade a release
helm upgrade monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --reuse-values              # keep existing values

# Upgrade and install if not exists
helm upgrade --install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace          # ← great for CI/CD pipelines


# ─── ROLLBACK ───────────────────────────────────────────
# See release history
helm history monitoring -n monitoring
# REVISION  STATUS     DESCRIPTION
# 1         superseded Install complete
# 2         deployed   Upgrade complete

# Rollback to previous version
helm rollback monitoring -n monitoring

# Rollback to specific revision
helm rollback monitoring 1 -n monitoring


# ─── DELETE ─────────────────────────────────────────────
helm uninstall monitoring -n monitoring
```

---

### 🔷 values.yaml — The Power of Helm

Instead of editing templates, you override values:

```yaml
# my-values.yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 250m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```bash
# Install with your values
helm install my-app bitnami/nginx \
  -f my-values.yaml \
  -n production

# Different values per environment
helm install my-app bitnami/nginx \
  -f base-values.yaml \
  -f prod-values.yaml \      # prod values override base
  -n production
```

---

### 🔷 See Default Values First

```bash
# See ALL configurable values for a chart
helm show values prometheus-community/kube-prometheus-stack

# Save to file and customize
helm show values prometheus-community/kube-prometheus-stack \
  > default-values.yaml

# Edit what you need → use as your values file
```

---

### 🛠️ Create Your Own Helm Chart

```bash
# Scaffold a new chart
helm create my-app

# Structure created:
# my-app/
# ├── Chart.yaml           ← chart metadata
# ├── values.yaml          ← default values
# ├── charts/              ← dependencies
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     ├── hpa.yaml
#     ├── serviceaccount.yaml
#     ├── NOTES.txt        ← shown after install
#     └── _helpers.tpl     ← reusable template functions
```

---

### 🔷 Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
description: My Django application Helm chart
type: application
version: 0.1.0           # chart version (your version)
appVersion: "2.1.0"      # app version (your app's version)

dependencies:            # other charts this depends on
  - name: mysql
    version: "9.10.0"
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled
```

---

### 🔷 Template Syntax — Go Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}     # ← helper function
  namespace: {{ .Release.Namespace }}          # ← built-in value
  labels:
    app: {{ .Chart.Name }}
    version: {{ .Chart.AppVersion }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}         # ← from values.yaml
  selector:
    matchLabels:
      app: {{ include "my-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "my-app.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}

          # Conditional block
          {{- if .Values.resources }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
```

---

### 🔷 Built-in Helm Objects

```
.Release.Name        → name you gave the release
.Release.Namespace   → namespace it's installed in
.Release.IsInstall   → true if first install
.Release.IsUpgrade   → true if upgrade

.Chart.Name          → chart name from Chart.yaml
.Chart.Version       → chart version
.Chart.AppVersion    → app version

.Values.xxx          → from your values.yaml

.Files.Get "config.json"  → read a file from chart
```

---

### 🛠️ Helm Dry Run & Debug

```bash
# Dry run — see what would be applied WITHOUT applying
helm install my-app ./my-app \
  --dry-run \
  --debug \
  -f my-values.yaml

# Template — render templates locally
helm template my-app ./my-app \
  -f my-values.yaml

# Lint — check chart for errors
helm lint ./my-app

# Diff — see what would change (needs helm-diff plugin)
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-app ./my-app -f my-values.yaml
```

---

## 🔷 PART 2 — Custom Resource Definitions (CRDs)

### What is a CRD?

```
K8s ships with built-in resource types:
  Pod, Deployment, Service, ConfigMap, Secret...

CRDs let you ADD YOUR OWN resource types to k8s.

Examples of custom resources:
  Prometheus    → added ServiceMonitor, PrometheusRule
  Cert-manager  → added Certificate, ClusterIssuer
  Istio         → added VirtualService, DestinationRule
  ArgoCD        → added Application, AppProject
  Kafka         → added KafkaTopic, KafkaUser

After CRD is installed:
  kubectl get servicemonitors     ← works! ✅
  kubectl get certificates        ← works! ✅
  kubectl get kafkatopics         ← works! ✅
```

---

### 🔷 How CRDs Work

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. CRD installed                                           │
│     → k8s API learns about new resource type               │
│     → kubectl get, apply, delete now work on it            │
│                                                             │
│  2. You create a Custom Resource (CR) instance             │
│     → stored in etcd like any other resource               │
│                                                             │
│  3. An Operator (controller) watches for CRs               │
│     → takes action based on CR spec                        │
│     → reconciles actual state with desired state           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 🛠️ Define a CRD

```yaml
# crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com        # plural.group
spec:
  group: mycompany.com                 # API group
  versions:
    - name: v1
      served: true                     # this version is active
      storage: true                    # stored in etcd
      schema:
        openAPIV3Schema:               # validation schema
          type: object
          properties:
            spec:
              type: object
              required: ["engine", "storage"]
              properties:
                engine:
                  type: string
                  enum: ["mysql", "postgres", "mongodb"]
                version:
                  type: string
                storage:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 5
  scope: Namespaced                    # Namespaced or Cluster
  names:
    plural: databases                  # kubectl get databases
    singular: database                 # kubectl get database
    kind: Database                     # YAML kind: Database
    shortNames:
      - db                             # kubectl get db
```

```bash
kubectl apply -f crd.yaml

# Now you have a new resource type!
kubectl get databases
kubectl get db
kubectl api-resources | grep mycompany
```

---

### 🛠️ Create a Custom Resource

```yaml
# my-database.yaml
apiVersion: mycompany.com/v1
kind: Database                         # our custom kind
metadata:
  name: production-db
  namespace: dev
spec:
  engine: mysql                        # validated against CRD schema
  version: "8.0"
  storage: "50Gi"
  replicas: 3
```

```bash
kubectl apply -f my-database.yaml
kubectl get databases -n dev
kubectl describe database production-db -n dev
```

> Without an Operator watching it, this CR just sits in etcd.
> The Operator is what makes it DO something.

---

## 🔷 PART 3 — Operators

### What is an Operator?

```
Operator = CRD + Controller that automates complex app management

The Operator Pattern:
  Encodes human operational knowledge into software

Human DBA knowledge:
  "To deploy MySQL cluster:
    1. Start primary node first
    2. Wait for primary to be ready
    3. Configure replication on secondaries
    4. Set up backup schedule
    5. Monitor replication lag
    6. Handle failover if primary dies
    7. Resize PVCs when storage is low"

Operator knowledge:
  Watch for Database CRs
  → do all the above AUTOMATICALLY ✅
```

---

### 🔷 Operator vs Plain Deployment

```
Plain Deployment of MySQL:
  You write YAML for StatefulSet, Services, PVCs
  You manually handle backup schedules
  You manually handle failover
  You manually handle upgrades
  You manually handle replication setup
  = lots of ongoing manual work

MySQL Operator:
  You write: kind: MysqlCluster
             replicas: 3
             storage: 50Gi
  Operator handles EVERYTHING else automatically ✅

This is why Operators are powerful for stateful apps.
```

---

### 🔷 Real Operators You'll Use

```
Database Operators:
  → Zalando PostgreSQL Operator
  → MySQL Operator (Oracle)
  → MongoDB Community Operator
  → Redis Operator

Messaging:
  → Strimzi (Kafka Operator)
  → RabbitMQ Operator

Security:
  → cert-manager (TLS certificate Operator)
  → Vault Operator

Monitoring:
  → Prometheus Operator (part of kube-prometheus-stack)

GitOps:
  → ArgoCD Operator

Ingress:
  → AWS Load Balancer Controller
  → nginx Ingress Operator
```

---

### 🛠️ cert-manager — Real Operator Example

```bash
# Install cert-manager (manages TLS certs automatically)
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Verify CRDs installed
kubectl get crds | grep cert-manager
# certificates.cert-manager.io           ✅
# clusterissuers.cert-manager.io         ✅
# issuers.cert-manager.io                ✅
```

```yaml
# Create a ClusterIssuer (tells cert-manager HOW to get certs)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer              # cert-manager CRD ✅
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
---
# Request a certificate
apiVersion: cert-manager.io/v1
kind: Certificate                # cert-manager CRD ✅
metadata:
  name: myapp-tls
  namespace: production
spec:
  secretName: myapp-tls-secret  # cert stored here
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
    - www.myapp.example.com
```

```
cert-manager Operator sees Certificate CR
  → calls Let's Encrypt API
  → completes ACME challenge
  → stores TLS cert in myapp-tls-secret
  → auto-renews before expiry ✅

You never touch certificate management again.
```

---

### 🛠️ Strimzi Kafka Operator — Another Example

```bash
# Install Strimzi Kafka Operator
helm repo add strimzi https://strimzi.io/charts/
helm install strimzi strimzi/strimzi-kafka-operator \
  --namespace kafka \
  --create-namespace
```

```yaml
# Deploy entire Kafka cluster with 3 lines of intent
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka                       # Strimzi CRD ✅
metadata:
  name: production-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3                   # 3 Kafka brokers
    storage:
      type: persistent-claim
      size: 100Gi
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3

  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi

  entityOperator:
    topicOperator: {}
    userOperator: {}
---
# Create a topic
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic                  # Strimzi CRD ✅
metadata:
  name: orders-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: production-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000       # 7 days
```

```
Strimzi Operator sees Kafka CR
  → creates ZooKeeper StatefulSet
  → creates Kafka broker StatefulSets
  → creates Services for each broker
  → configures inter-broker replication
  → manages rolling upgrades
  → handles broker failures

You get a production Kafka cluster
with 30 lines of YAML ✅
```

---

## 🔷 PART 4 — Kubernetes API

### K8s API Deep Dive

```
Everything in k8s is an API call.

kubectl get pods
  → GET /api/v1/namespaces/dev/pods

kubectl apply -f deployment.yaml
  → POST /apis/apps/v1/namespaces/dev/deployments

kubectl delete pod my-pod
  → DELETE /api/v1/namespaces/dev/pods/my-pod
```

---

### 🔷 API Groups & Versions

```
Core API (v1):
  /api/v1/pods
  /api/v1/services
  /api/v1/configmaps
  /api/v1/secrets
  /api/v1/nodes
  /api/v1/namespaces

Named API groups:
  /apis/apps/v1/deployments
  /apis/apps/v1/statefulsets
  /apis/batch/v1/jobs
  /apis/batch/v1/cronjobs
  /apis/networking.k8s.io/v1/ingresses
  /apis/autoscaling/v2/horizontalpodautoscalers
  /apis/rbac.authorization.k8s.io/v1/roles

CRD groups (custom):
  /apis/monitoring.coreos.com/v1/servicemonitors
  /apis/cert-manager.io/v1/certificates
  /apis/kafka.strimzi.io/v1beta2/kafkas
```

```bash
# Explore the API
kubectl api-resources          # all resource types
kubectl api-versions           # all API versions
kubectl explain pod            # docs for a resource
kubectl explain pod.spec       # docs for a field
kubectl explain deployment.spec.template.spec.containers
```

---

### 🛠️ Call K8s API Directly

```bash
# Get API server URL
kubectl cluster-info
# Kubernetes control plane: https://127.0.0.1:6443

# Use kubectl proxy to call API locally
kubectl proxy &

# Now call API directly
curl http://localhost:8001/api/v1/namespaces/dev/pods

# Or with token
TOKEN=$(kubectl create token admin-user -n kubernetes-dashboard)

curl -k https://127.0.0.1:6443/api/v1/namespaces/dev/pods \
  -H "Authorization: Bearer $TOKEN"
```

---

### 🔷 K8s API From Your App (Python)

```python
# Using official kubernetes Python client
# pip install kubernetes

from kubernetes import client, config

# Load kubeconfig
config.load_incluster_config()    # inside cluster (pod)
# OR
config.load_kube_config()         # local development

# Create API client
v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

# List pods
pods = v1.list_namespaced_pod(namespace="dev")
for pod in pods.items:
    print(f"{pod.metadata.name}: {pod.status.phase}")

# Create deployment
deployment = client.V1Deployment(
    metadata=client.V1ObjectMeta(name="my-app"),
    spec=client.V1DeploymentSpec(
        replicas=3,
        selector=client.V1LabelSelector(
            match_labels={"app": "my-app"}
        ),
        template=client.V1PodTemplateSpec(
            metadata=client.V1ObjectMeta(
                labels={"app": "my-app"}
            ),
            spec=client.V1PodSpec(
                containers=[
                    client.V1Container(
                        name="app",
                        image="nginx:1.25"
                    )
                ]
            )
        )
    )
)

apps_v1.create_namespaced_deployment(
    namespace="dev",
    body=deployment
)
```

---

## ✅ Chapter 15 Summary

| Concept | Key Point |
|---|---|
| Helm | Package manager for k8s — charts, releases, values |
| Chart | Package of k8s manifests with templating |
| Release | Installed instance of a chart |
| values.yaml | Configuration that customizes a chart |
| helm upgrade --install | Idempotent — perfect for CI/CD |
| CRD | Extends k8s API with custom resource types |
| Custom Resource | Instance of a CRD — stored in etcd |
| Operator | CRD + Controller that automates app management |
| cert-manager | TLS certificate Operator — auto-renews certs |
| Strimzi | Kafka Operator — full cluster from simple YAML |

---

## 🧠 Interview Questions You Can Now Nail

1. **"What is Helm and why do we use it?"**
   → Helm is a package manager for k8s. It bundles complex multi-resource deployments into reusable charts, supports templating for different environments, and tracks releases for easy upgrades and rollbacks.

2. **"What is a CRD?"**
   → Custom Resource Definition extends the k8s API with new resource types. Tools like Prometheus, cert-manager, and Istio add their own resource types via CRDs so you manage them with native kubectl commands.

3. **"What is a Kubernetes Operator?"**
   → An Operator encodes operational knowledge about a specific application into a controller. It watches for Custom Resources and automates complex tasks like provisioning databases, handling failover, managing backups, and auto-renewing certificates.

4. **"What's the difference between helm install and helm upgrade --install?"**
   → `helm install` fails if the release already exists. `helm upgrade --install` installs if it doesn't exist, upgrades if it does — idempotent and perfect for CI/CD pipelines.

5. **"How does cert-manager work?"**
   → cert-manager is an Operator that introduces Certificate and ClusterIssuer CRDs. You declare a Certificate resource specifying the domain and issuer. cert-manager automatically calls Let's Encrypt, completes the ACME challenge, stores the TLS cert as a k8s Secret, and auto-renews before expiry.

---

## 🔷 Where We Are in the Roadmap

```
✅ Chapter 1  — Introduction & K8s Concepts
✅ Chapter 2  — K8s Architecture
✅ Chapter 3  — Cluster Setup (KIND, Minikube, Kubeadm)
✅ Chapter 4  — Namespaces & Pods
✅ Chapter 5  — Deployments, ReplicaSets & Rolling Updates
✅ Chapter 6  — Jobs, CronJobs & StatefulSets
✅ Chapter 7  — Storage (PV, PVC, StorageClasses)
✅ Chapter 8  — Services & Networking
✅ Chapter 9  — ConfigMaps & Secrets
✅ Chapter 10 — Resource Quotas, Limits & Probes
✅ Chapter 11 — Taints, Tolerations & Node Affinity
✅ Chapter 12 — HPA & VPA
✅ Chapter 13 — RBAC
✅ Chapter 14 — Monitoring & Logging
✅ Chapter 15 — Helm, CRDs & Operators

🔜 Chapter 16 — Sidecar & Init Containers
🔜 Chapter 17 — Istio Service Mesh
🔜 Chapter 18 — Projects (3-tier Chat App + EKS Mega Project)
```

---

Ready for **Chapter 16 → Sidecar & Init Containers** — the container patterns that power logging, security, proxies, and app initialization in production? Say **"next"** 🚀
